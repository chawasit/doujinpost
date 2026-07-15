# Spec — Authentication (Password · Google Sign-In · Passkeys)

Status: draft · Owner: platform · Depends on: `PLAN.md` §7, `specs/user-verification.md`
Runtime: **Bun + ElysiaJS** (TypeScript)

One account can carry **multiple credentials**: a password, a linked Google
account, and one or more passkeys. Any of them authenticates into the same
server-side session. Passkeys are the preferred method and can act as a second
factor; Google is the low-friction on-ramp; password is the always-available
fallback.

---

## 1. Goals & non-goals

**Goals**
- Sign up / sign in with **password**, **Google (OIDC)**, or **passkey (WebAuthn)**.
- Let one user link several credentials and manage them.
- Phishing-resistant login via passkeys; keep a recovery path.
- Session model that works for a **PWA** (installable, offline-capable) without
  exposing long-lived tokens to JS.

**Non-goals**
- KYC / identity proofing (out of scope; see `PLAN.md` §7).
- Contact-channel proof — that's `specs/user-verification.md`; auth and
  verification are distinct (a Google login proves email *ownership by Google*,
  which we accept as email-verified only when `email_verified=true` in the token).
- Other social IdPs for v1 (Apple/Discord can follow the same OIDC adapter).

---

## 2. Session model (shared by all methods)

- **Server-side sessions**, not JWTs in local storage. On success we create a
  `session` row and set an opaque, random session id in a cookie:
  `Secure; HttpOnly; SameSite=Lax; Path=/`. A short-lived CSRF token is issued
  separately for state-changing requests.
- Sliding expiry (e.g. idle 14d, absolute 90d); rotate id on privilege change
  (login, 2FA pass, password change) to prevent fixation.
- **PWA note**: the service worker never sees the session cookie (HttpOnly), and
  must not cache authenticated API responses. Offline reader content is limited
  to already-downloaded, entitlement-checked media; the app treats "no session"
  as read-only-public and re-auths on reconnect.
- Sessions are enumerable and revocable in `GET /me/sessions` /
  `DELETE /me/sessions/:id` (device, ip-trunc, last-seen).

```sql
users (id uuid pk, handle text unique, primary_email text,
       email_verified bool, state text, created_at timestamptz)

credentials (                      -- one row per credential, any type
  id uuid pk, user_id uuid fk,
  type text check (type in ('password','oauth_google','passkey')),
  -- password:
  pw_hash bytea,
  -- oauth:
  provider_sub text,               -- Google 'sub' (stable user id)
  -- passkey (WebAuthn):
  cred_id bytea,                   -- credential id (unique)
  public_key bytea, sign_count bigint, aaguid bytea, transports text[],
  label text, created_at timestamptz, last_used_at timestamptz,
  unique (type, provider_sub),
  unique (cred_id)
)

sessions (id bytea pk, user_id uuid fk, amr text[],  -- methods used
          created_at, last_seen_at, absolute_exp, ip_trunc, ua_hash)

webauthn_challenges (id uuid pk, user_id uuid, challenge bytea,
                     purpose text, expires_at)   -- transient, one-shot
oauth_states (state text pk, nonce text, pkce_verifier text,
              redirect_to text, expires_at)      -- transient, one-shot
```

`amr` (authentication methods references) records how the session was proven
(`pwd`, `google`, `passkey`, `otp`) so policy can require step-up for sensitive
actions.

---

## 3. Password (baseline)

- Argon2id (Bun-native binding), per-user salt, tuned params; verify in constant
  time. Breach-password check (k-anonymity range API) on set.
- Login throttled per-account and per-IP (Redis sliding window); generic error
  on bad credentials (no user-enumeration).
- Optional TOTP 2FA: if enabled, password login requires a valid code before the
  session is fully privileged (`amr` gets `otp`).
- Reset via emailed single-use, TTL-bound signed token (reuses the OTP/token
  machinery from `specs/user-verification.md`).

---

## 4. Google Sign-In (OpenID Connect, Authorization Code + PKCE)

Server-side Authorization Code flow with PKCE — **not** Google One-Tap ID tokens
validated client-side, so the exchange and account linking happen on the server.

```
GET /auth/google/start
  → generate state + nonce + PKCE (verifier/challenge); persist oauth_states
  → 302 to Google authorize endpoint (scope: openid email profile)

GET /auth/google/callback?code&state
  → validate state (CSRF), load one-shot oauth_states row
  → exchange code + pkce_verifier for tokens at Google's token endpoint
  → verify id_token: signature (JWKS), iss, aud, exp, nonce
  → extract sub, email, email_verified
  → link-or-create (see §4.1) → create session (amr += 'google')
```

### 4.1 Account linking rules

- Match on **`provider_sub`** first (stable). If found → log that user in.
- Else if a user exists with the **same verified email** → **do not silently
  merge.** Require the user to prove the existing account (password or passkey)
  before attaching the Google credential — prevents account takeover via an
  email an attacker got Google to assert. If they can't, offer support recovery.
- Else → create a new user; set `email_verified` from the token's
  `email_verified` claim (so a Google user reaches verification level 1 without a
  separate email OTP).
- Users can link/unlink Google in settings; unlinking is blocked if it would
  leave the account with **zero usable credentials**.

---

## 5. Passkeys (WebAuthn / FIDO2)

Uses the W3C WebAuthn ceremonies via a server library
(`@simplewebauthn/server`), Elysia routes issuing/verifying challenges.

**Relying Party**: `rp.id` = registrable domain; `rp.origin` pinned to the PWA
origin. `userVerification: 'preferred'`, `residentKey: 'preferred'` (so passkeys
are discoverable → usernameless login).

### 5.1 Registration (add a passkey to an account)

```
POST /auth/passkey/register/start   (session required)
  → create webauthn_challenge(purpose='register'); return PublicKeyCredentialCreationOptions
    (excludeCredentials = the user's existing cred_ids)
POST /auth/passkey/register/finish  { attestation }
  → verify challenge, origin, rp.id; store cred_id, public_key, sign_count,
    aaguid, transports; label the device → 201
```

### 5.2 Authentication (login with a passkey)

```
POST /auth/passkey/login/start   { handle? }     (no session)
  → challenge; allowCredentials from handle if given, else empty (discoverable)
POST /auth/passkey/login/finish  { assertion }
  → find credential by cred_id, verify signature against public_key,
    verify challenge + origin + rp.id,
    check sign_count strictly increases (clone detection) → update it
  → create session (amr += 'passkey')
```

- **Clone detection**: a non-increasing `sign_count` (when the authenticator
  reports one) flags a possibly cloned key → refuse + alert.
- Passkeys satisfy **both factors** (possession + user verification), so a
  passkey login needs no separate TOTP step.
- **Step-up**: sensitive actions (change email, delete account, disable 2FA) can
  demand a fresh passkey/TOTP assertion even within an active session.

---

## 6. Recovery & anti-lockout

Multiple credentials are the primary recovery story, but we still guarantee a
way back in:

- On first passkey/Google-only signup, prompt to add a **second** credential
  (another passkey or a password) — warn before leaving a single point of failure.
- **Recovery codes**: one-time backup codes generated when 2FA/passkey-only is
  enabled; hashed at rest.
- Email-based reset (password) and support-assisted recovery (identity signals +
  cooldown) as last resorts.
- **Deletion guard**: any credential-removal endpoint refuses if it would leave
  the account with no usable sign-in method.

---

## 7. Endpoints summary

| Method | Path | Auth | Purpose |
|-------|------|------|---------|
| POST | `/auth/password/register` | none | create account (password) |
| POST | `/auth/password/login`    | none | password (+TOTP) login |
| POST | `/auth/password/reset/*`  | token | reset flow |
| GET  | `/auth/google/start`      | none | begin OIDC |
| GET  | `/auth/google/callback`   | none | finish OIDC → session |
| POST | `/auth/passkey/register/start\|finish` | session | add passkey |
| POST | `/auth/passkey/login/start\|finish`    | none | passkey login |
| POST | `/auth/logout`            | session | revoke current session |
| GET  | `/me/credentials`         | session | list linked methods |
| DELETE | `/me/credentials/:id`   | session | unlink (guarded) |
| GET/DELETE | `/me/sessions[/:id]` | session | list / revoke sessions |

All state-changing routes are CSRF-protected and rate-limited; all challenge/
state rows are single-use and TTL-bound.

---

## 8. Security notes

- OIDC: PKCE + `state` (CSRF) + `nonce` (replay); verify `iss`/`aud`/`exp` and
  the JWKS signature server-side; never trust a client-presented ID token alone.
- WebAuthn: verify `origin` and `rp.id`, one-shot challenges, strict sign-count,
  store only public keys (no shared secret to leak).
- Sessions: opaque ids, HttpOnly/Secure/SameSite, rotation on privilege change,
  server-side revocation.
- Linking never merges accounts on an unproven email match (§4.1).
- `amr` drives step-up authZ for destructive actions.
- Interplay with verification: Google `email_verified=true` grants level 1;
  passkey/password signups still run the email OTP from
  `specs/user-verification.md` to reach level 1.

---

## 9. Roadmap fit

**`PLAN.md` Phase 0** (auth foundation): ship password + Google + passkey +
sessions together — they share the session/credential core, so building them
piecemeal costs more than doing them once. TOTP and recovery codes land in the
same phase; step-up policy is wired as sensitive endpoints appear in later phases.

**Test checklist**: password login + lockout; TOTP gate; Google new-user create;
Google existing-`sub` login; Google email-collision requires proof (no silent
merge); passkey register + discoverable login; sign-count clone rejection;
credential-removal guard (no zero-credential accounts); session rotation on
privilege change; PWA offline → reconnect re-auth.
