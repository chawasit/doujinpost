# Spec — Basic User Verification (Email + SMS)

Status: draft · Owner: platform · Depends on: `PLAN.md` §7 (auth)

Verifies that an account controls an **email address** and/or a **phone number**
before granting trust-gated abilities (uploading, downloading, posting). Both
channels use the same one-time-code (OTP) engine and the same rate/abuse rules;
only the delivery transport differs.

---

## 1. Goals & non-goals

**Goals**
- Prove control of an email and/or phone via a short-lived code.
- Gate sensitive actions behind a verification level.
- Be resilient to spam, code-guessing, and delivery-cost abuse (SMS pumping).
- Provider-agnostic delivery (swap SES/SendGrid, Twilio/Vonage without app changes).

**Non-goals**
- Full KYC / identity documents. This is control-of-channel only.
- Passwordless login (email is a *verification* factor here, not a login method).
  TOTP 2FA stays as specified in `PLAN.md`; this spec is orthogonal to it.
- Age verification (separate, jurisdiction-specific — see `PLAN.md` §7).

---

## 2. Verification levels

Each account has a `verification_level`, the max of what it has proven:

| Level | Meaning                        | Unlocks |
|------:|--------------------------------|---------|
| `0`   | unverified (just registered)   | browse only |
| `1`   | email verified                 | comment, forum post, download |
| `2`   | email **and** phone verified   | upload works, higher rate limits |

Levels are additive capabilities, configurable per action in one policy table —
so the level→action mapping can change without code edits.

---

## 3. Data model

```sql
-- one row per (user, channel, address); a user may have several emails/phones
contact_methods (
  id            uuid pk,
  user_id       uuid fk -> users(id),
  channel       text check (channel in ('email','sms')),
  address       text not null,          -- email or E.164 phone
  address_norm  text not null,          -- lowercased email / normalised E.164
  is_primary    boolean default false,
  verified_at   timestamptz,            -- null = unverified
  created_at    timestamptz default now(),
  unique (channel, address_norm, user_id)
);

-- transient challenge; code stored only as a hash
verification_challenges (
  id             uuid pk,
  contact_id     uuid fk -> contact_methods(id),
  code_hash      bytea not null,        -- HMAC/argon2 of the OTP, never plaintext
  purpose        text,                  -- 'verify_email' | 'verify_phone'
  expires_at     timestamptz not null,
  attempts       int default 0,
  max_attempts   int default 5,
  consumed_at    timestamptz,
  created_at     timestamptz default now()
);
```

- **Global uniqueness rule**: a given `address_norm` may be *verified* on only one
  account at a time (partial unique index where `verified_at is not null`).
  Unverified duplicates are allowed so typos/abandoned attempts don't lock an
  address forever.
- Codes are never stored in plaintext — only `code_hash`. On check we hash the
  submitted code and compare in constant time.

---

## 4. OTP engine

- **Email**: 6-digit numeric OR a signed magic link (see §5). Default 6-digit.
- **SMS**: 6-digit numeric only (links are unreliable in SMS).
- **Entropy**: 6 digits = 1e6 space; combined with ≤5 attempts + short TTL +
  rate limits this is safe. Codes drawn from a CSPRNG, not `random`.
- **TTL**: 10 minutes (email), 5 minutes (SMS). Configurable.
- **Single active challenge** per (contact, purpose): issuing a new code
  invalidates the prior one (`consumed_at = now()` on the old row).
- **Attempt lockout**: after `max_attempts` wrong submissions the challenge is
  burned; user must request a new code.

---

## 5. Flows

### 5.1 Email verification

```
POST /verify/email/start   { email }            (auth required)
  → normalise, upsert contact_method (channel=email)
  → reject if already verified on another account
  → rate-limit check (§6)
  → mint challenge, enqueue send via provider
  → 202 { challenge_id, expires_at }            (never reveal the code)

POST /verify/email/confirm { challenge_id, code }
  → constant-time compare hash; check TTL, attempts, not consumed
  → on success: contact_method.verified_at = now(); recompute level
  → 200 { verification_level }
```

**Magic-link variant**: the email may embed
`GET /verify/email/link?token=<signed>` where `token` is an HMAC-signed,
single-use, TTL-bound blob encoding `challenge_id`. Clicking confirms without
manual code entry. Same TTL/attempt/consume semantics; link is one-shot.

### 5.2 SMS verification

```
POST /verify/phone/start   { phone }            (auth required)
  → parse to E.164, validate line type (reject VoIP/premium if policy says so)
  → upsert contact_method (channel=sms)
  → reject if verified elsewhere; SMS-specific rate + cost caps (§6)
  → mint 6-digit challenge, enqueue SMS send
  → 202 { challenge_id, expires_at }

POST /verify/phone/confirm { challenge_id, code }
  → same confirm logic as email → level recomputed
```

### 5.3 Resend

```
POST /verify/{channel}/resend { challenge_id }
  → allowed only after a cooldown (e.g. 60s); counts against hourly cap
  → invalidates old challenge, issues a fresh code+TTL
```

---

## 6. Rate limiting & abuse controls

Layered, because email and SMS have different abuse profiles (SMS costs money —
**SMS pumping / toll fraud** is the real threat).

| Control | Email | SMS |
|--------|-------|-----|
| Per-account sends / hour | 5 | 3 |
| Per-account sends / day  | 20 | 5 |
| Per-IP sends / hour      | 10 | 5 |
| Resend cooldown          | 60s | 60s |
| Per-address lifetime attempts before hard block | 20 | 10 |
| Confirm attempts / challenge | 5 | 5 |

Additional SMS defences:
- **Geo/prefix allowlist or risk-scoring** — block or step-up-challenge country
  codes with high fraud rates unless the account has other trust signals.
- **Line-type check** — optionally reject VoIP/disposable numbers via the
  provider lookup API.
- **Global spend circuit-breaker** — if SMS send volume exceeds a rolling
  threshold, auto-throttle and alert (stops a pumping attack from running up the bill).
- **Enumeration safety** — `start` responses never reveal whether an address is
  already registered; they always return `202` with a generic body.

Counters live in Redis (sliding window). Challenges + verified state live in
Postgres (durable).

---

## 7. Provider abstraction

```python
class Notifier(Protocol):
    def send_email(self, to: str, template: str, ctx: dict) -> SendResult: ...
    def send_sms(self, to_e164: str, body: str) -> SendResult: ...
```

- Concrete adapters: `SesNotifier` / `SendgridNotifier`, `TwilioNotifier` /
  `VonageNotifier`. Selected by config; no call site knows the vendor.
- Sends run in the **worker queue**, not inline in the request, with retry +
  dead-letter. A `SendResult` (provider id, status) is logged for delivery
  auditing and cost tracking.
- Templates versioned; localisation-ready (user locale → template variant).
- Secrets (API keys) in KMS/secret store, never in the DB or repo.

---

## 8. Endpoints summary

| Method | Path | Auth | Purpose |
|-------|------|------|---------|
| POST | `/verify/email/start`    | session | issue email code |
| POST | `/verify/email/confirm`  | session | confirm email code |
| GET  | `/verify/email/link`     | token   | magic-link confirm |
| POST | `/verify/phone/start`    | session | issue SMS code |
| POST | `/verify/phone/confirm`  | session | confirm SMS code |
| POST | `/verify/{channel}/resend` | session | resend within limits |
| GET  | `/me/verification`       | session | current level + methods |
| DELETE | `/me/contact/{id}`     | session | remove an unused contact method |

All mutating endpoints require an authenticated session and are CSRF-protected;
all are rate-limited per §6.

---

## 9. Security notes

- Codes: CSPRNG, hashed at rest, constant-time compare, short TTL, capped attempts.
- No plaintext code or verification state in logs; log `challenge_id` only.
- Magic-link and API responses are enumeration-safe (generic `202`).
- Changing/removing a verified primary contact re-checks whether the account
  still meets any level gate and may drop its level.
- Verifying an address already verified elsewhere is refused (§3 uniqueness),
  preventing one phone/email from vouching for many accounts.
- Ties into moderation: verification level is a signal in the spam heuristic and
  in DMCA repeat-infringer accounting (`PLAN.md` §4, §5).

---

## 10. Roadmap fit

Slots into **`PLAN.md` Phase 0–1** (auth + first trust gates). Email verification
(level 1) ships in Phase 0 alongside auth; SMS (level 2) can follow in Phase 1
once the upload path — the action it gates — exists.

**Test checklist**
- happy path email + SMS; expired code; wrong code lockout; resend cooldown;
  duplicate-address refusal; enumeration-safety of `start`; SMS rate/spend caps;
  provider failure → retry/dead-letter; level recompute on add/remove.
