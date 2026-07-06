# Authentication Technologies — Deep Reference

A structured breakdown of the major auth mechanisms used in modern full-stack / SaaS systems, framed for system design decision-making.

---

## 1. Session-Based Authentication (Cookie + Server Session Store)

**Technologies / npm packages:** `express-session`, `next-auth` (database session strategy), `iron-session`, Redis (`connect-redis`) as session store, PostgreSQL for persistent sessions.

**Pros:**
1. Server has full control — instant revocation (just delete the session row/key).
2. Simple mental model: session ID in an `httpOnly` cookie, no client-side token parsing.
3. Small payload sent per request (just a session ID, not a full claims object).

**Cons:**
1. Requires a stateful store (Redis/DB) — adds infra dependency and a network hop per request.
2. Harder to scale horizontally without a shared session store (sticky sessions are a hack, not a fix).
3. CSRF exposure since cookies are auto-sent by browsers — needs CSRF tokens or `SameSite` config.

**Benefits:**
1. Best-in-class for **revocation and forced logout** (e.g., "log out of all devices").
2. Works naturally with server-rendered apps (Next.js SSR, traditional MVC).

**Use case scenarios:**
- *Explanatory:* Any app where you control both frontend and backend, are within one domain/subdomain family, and revocation matters (banking, admin dashboards, internal tools).
- *Expected:* Traditional web apps, B2B dashboards, anything with strict "kill session now" compliance requirements (SOC2, HIPAA-adjacent tooling).

**Senior system design thought:**
Session auth trades scalability for control. In a multi-tenant SaaS, this is often the safer default for admin/back-office surfaces because you need deterministic revocation — a compromised admin account must be killable in real time, and JWTs can't do that without a blocklist (which just reintroduces state anyway). Pair it with Redis with TTL matching your session expiry, and put the session store behind the same per-tenant isolation boundary as the rest of your data.

**Example apps:** GitHub (web sessions), most Rails/Django admin panels, banking portals.

---

## 2. JWT (JSON Web Tokens)

**Technologies / npm packages:** `jsonwebtoken`, `jose` (modern, actively maintained, supports JWK rotation), `@nestjs/jwt`, `passport-jwt`.

**Pros:**
1. Stateless — no DB lookup needed to validate a request, just signature verification.
2. Self-contained claims (user id, roles, tenant id) reduce extra queries per request.
3. Works well across services/domains without a shared session store — ideal for microservices.

**Cons:**
1. Revocation is hard — a valid JWT is valid until expiry unless you build a blocklist (which reintroduces state, defeating the purpose).
2. Token bloat if you stuff too many claims in — larger payload on every request.
3. If the signing secret leaks or you use `alg: none`/weak algorithms, you get full auth bypass — a whole CVE category (`jwt-none-algorithm` attacks).

**Benefits:**
1. Enables **stateless horizontal scaling** — any node can verify a request independently.
2. Natural fit for service-to-service auth and API-first architectures.

**Use case scenarios:**
- *Explanatory:* Distributed systems where the auth server and resource servers are decoupled, mobile apps, public APIs.
- *Expected:* Access tokens with short TTL (5–15 min) + refresh token rotation, microservice-to-microservice calls, mobile app sessions.

**Senior system design thought:**
The real design decision isn't "JWT vs sessions," it's "where do you want your state — in the token or in your store." Short-lived JWT access tokens (5–15 min) + a long-lived, revocable refresh token in a DB/Redis gives you the stateless scaling benefit for 95% of requests while keeping a real revocation lever for the 5% that matters (logout, compromised account). This hybrid is what most production systems (including likelife.ai-style multi-tenant SaaS) actually run — pure stateless JWT auth without a refresh/revocation layer is a common junior mistake that bites during incident response.

**Example apps:** Auth0-issued tokens, most REST/GraphQL APIs, Slack API tokens (a variant).

---

## 3. OAuth 2.0

**Technologies / npm packages:** `passport-oauth2`, `simple-oauth2`, `next-auth` (providers config), `openid-client`.

**Pros:**
1. Delegated authorization — users grant scoped access without sharing passwords.
2. Industry-standard flows (Authorization Code + PKCE) are battle-tested against common attacks.
3. Decouples "who can access this resource" from "who is this user" (that's OIDC's job — see below).

**Cons:**
1. OAuth2 is an **authorization** framework, not authentication — using it for login without OIDC on top is a classic misuse (this was literally the reason OIDC was created).
2. Flow complexity — Authorization Code, Client Credentials, Device Code, PKCE — picking the wrong grant type for your use case creates real vulnerabilities.
3. Redirect-based flows are awkward in native mobile/desktop apps without PKCE and custom URI schemes.

**Benefits:**
1. Lets you integrate with third-party providers (Google, GitHub, Amazon) without ever touching user credentials.
2. Scoped tokens limit blast radius — a leaked token only grants what it was scoped for.

**Use case scenarios:**
- *Explanatory:* "Sign in with Google," third-party API integrations (e.g., your app connecting to a user's Google Calendar), machine-to-machine access grants.
- *Expected:* Authorization Code + PKCE for web/mobile login flows, Client Credentials grant for server-to-server integrations.

**Senior system design thought:**
The design smell to watch for: teams implementing raw OAuth2 for "login" and inventing their own ID-token equivalent. That's reinventing OIDC badly. If you're building a voice-agent SaaS platform that needs to connect into third-party calendars/CRMs, OAuth2 (Authorization Code + PKCE, with refresh token rotation) is exactly the right tool for the *integration* layer — but your own user login should ride on OIDC or a plain session/JWT scheme, not OAuth2 directly.

**Example apps:** "Continue with Google" buttons, Zapier/Make integrations, Slack app installs.

---

## 4. OpenID Connect (OIDC)

**Technologies / npm packages:** `openid-client`, `next-auth` (most providers are OIDC under the hood), `passport-openidconnect`.

**Pros:**
1. Standardized identity layer on top of OAuth2 — gives you a verifiable ID token (JWT) with user identity claims.
2. Interoperable across providers (Google, Microsoft, Okta, Auth0) with a consistent discovery mechanism (`/.well-known/openid-configuration`).
3. Built-in support for things like `nonce` and `id_token` validation to prevent replay attacks.

**Cons:**
1. Yet another layer of spec complexity on top of OAuth2 — steep learning curve to get right.
2. Token validation requires JWKS fetching/caching — an extra moving part (key rotation handling).
3. Overkill for a simple single-tenant internal tool with one login method.

**Benefits:**
1. Single Sign-On (SSO) across multiple apps/tenants becomes straightforward.
2. Federated identity — enterprise customers can bring their own IdP (Okta, Azure AD) and you don't manage their passwords at all.

**Use case scenarios:**
- *Explanatory:* Enterprise SaaS requiring "Login with your company SSO," consumer apps offering multi-provider login.
- *Expected:* B2B SaaS platforms where enterprise customers demand SAML/OIDC SSO as a contract requirement before they'll sign.

**Senior system design thought:**
For a B2B multi-tenant SaaS, OIDC support is often a **sales-blocking feature** — enterprise deals stall without it. Design your user/session model early so `tenant_id` and `identity_provider` are first-class fields, not retrofitted later. Mixing OIDC-issued identity with your own internal session/JWT layer (i.e., OIDC only for the login handshake, then you issue your own app session) is the cleanest separation of concerns.

**Example apps:** "Sign in with Microsoft" enterprise SSO, Okta-integrated dashboards, Google Workspace login.

---

## 5. SAML (Security Assertion Markup Language)

**Technologies / npm packages:** `passport-saml`, `samlify`, `node-saml`.

**Pros:**
1. Deeply entrenched in enterprise IT — most large orgs' IdPs (Okta, ADFS, PingFederate) support it natively.
2. XML-signed assertions provide strong, auditable trust guarantees.
3. Mature, well-understood by enterprise security teams (easier procurement conversations).

**Cons:**
1. XML-based — verbose, harder to debug than JSON/JWT, and historically prone to signature-wrapping vulnerabilities if libraries are implemented sloppily.
2. No native support for mobile/native app flows — it's fundamentally a browser-redirect protocol.
3. Considered legacy next to OIDC for new builds — you mostly support it because enterprise customers demand it, not because it's technically nicer.

**Benefits:**
1. Unlocks enterprise contracts — many large companies' procurement/security teams simply require SAML SSO.
2. Strong assertion-signing model reduces token forgery risk when implemented correctly.

**Use case scenarios:**
- *Explanatory:* Enterprise B2B SaaS onboarding large customers with existing identity infrastructure.
- *Expected:* "SSO" checkbox in a SaaS pricing tier (usually gated behind an "Enterprise" plan).

**Senior system design thought:**
SAML is rarely a "want," it's a "must-have for this specific enterprise deal." The pragmatic approach used by most SaaS teams: don't hand-roll SAML — use a managed identity gateway (WorkOS, Auth0, Okta's SAML bridge) that normalizes SAML **and** OIDC into one internal interface, so your app code only ever talks to one abstraction regardless of what the customer's IdP speaks.

**Example apps:** Enterprise tools like Salesforce, Workday, and most "Enterprise SSO" tiers of SaaS products.

---

## 6. API Keys

**Technologies / npm packages:** custom implementation (hash + store, e.g., `bcrypt`/`argon2` for hashing keys), `express-rate-limit` for pairing with throttling.

**Pros:**
1. Dead simple to implement and integrate — one header, one lookup.
2. Easy to scope per-key (read-only vs write, per-endpoint permissions).
3. No session/token expiry complexity — works well for long-lived machine clients.

**Cons:**
1. No built-in expiry or rotation unless you build it yourself — keys tend to live forever and get leaked in git history/logs.
2. No user-identity context — just "this key can do X," not "this specific human did X" (weak audit trail).
3. Static secret transmitted on every request — higher exposure risk than short-lived tokens.

**Benefits:**
1. Ideal for server-to-server and third-party integration authentication.
2. Simple to rate-limit and monitor per key for abuse detection.

**Use case scenarios:**
- *Explanatory:* Public APIs (Stripe, Twilio-style), webhook senders authenticating to your endpoint, CI/CD pipelines calling your backend.
- *Expected:* `Authorization: Bearer sk_live_...` style keys with prefix-based key types (test/live) for safety.

**Senior system design thought:**
For a voice-agent or dev-tooling SaaS platform exposing a public API, API keys are non-negotiable as your primary external auth mechanism — customers integrating programmatically don't want an OAuth dance for a backend cron job. The senior move: store only a hash of the key (like a password), show the plaintext once at creation, support key rotation without downtime (allow two active keys per tenant during rotation windows), and scope keys to specific permissions rather than full-account access.

**Example apps:** Stripe, Twilio, OpenAI/Anthropic API keys, SendGrid.

---

## 7. Basic Auth / Digest Auth

**Technologies / npm packages:** `basic-auth` (Express middleware), built into most HTTP servers natively.

**Pros:**
1. Zero setup — supported natively by every HTTP client and browser.
2. Good enough for quick internal tooling or staging environment gating.

**Cons:**
1. Credentials sent (base64-encoded, **not encrypted**) on every request — mandatory HTTPS, and even then it's a weak model.
2. No session/token lifecycle — no expiry, no scoping, no revocation without changing the password.
3. Poor UX — browser's native auth prompt looks broken/untrustworthy to end users.

**Benefits:**
1. Fastest possible way to gate an internal/staging endpoint from accidental public exposure.

**Use case scenarios:**
- *Explanatory:* Staging environments, internal admin tools behind a VPN, quick prototype gating.
- *Expected:* Never for production user-facing auth — almost always a temporary/internal measure.

**Senior system design thought:**
If you see Basic Auth anywhere near a production user-facing flow, that's a red flag in a design review. Its only legitimate modern use is gating non-critical internal/staging surfaces where the cost of building real auth isn't justified yet.

**Example apps:** Staging site password gates, some legacy internal admin panels.

---

## 8. WebAuthn / Passkeys (FIDO2)

**Technologies / npm packages:** `@simplewebauthn/server` + `@simplewebauthn/browser`, `fido2-lib`.

**Pros:**
1. Phishing-resistant by design — credentials are bound to the origin, can't be tricked into use on a fake domain.
2. No shared secret to leak — public/private keypair, private key never leaves the device/secure enclave.
3. Great UX once adopted — biometric/PIN unlock instead of typing passwords.

**Cons:**
1. Device/platform fragmentation — cross-device passkey sync (via iCloud Keychain, Google Password Manager) is still maturing and inconsistent across ecosystems.
2. Account recovery is genuinely hard — if a user loses all their devices, you need a solid fallback (usually email + re-enrollment).
3. Requires meaningful frontend engineering investment vs. a simple password form.

**Benefits:**
1. Effectively eliminates credential-stuffing and phishing as attack vectors for enrolled users.
2. Reduces support burden long-term (no more "forgot password" resets once adoption is high).

**Use case scenarios:**
- *Explanatory:* Consumer apps wanting best-in-class security UX, high-value account protection (crypto exchanges, banking).
- *Expected:* Offered as an **option alongside** password/OTP, not yet a sole method for most consumer apps, due to recovery-flow gaps.

**Senior system design thought:**
Passkeys are the direction the industry is moving, but the system design nuance is the **recovery path**, not the happy path — that's where most WebAuthn rollouts fail in practice. Design your user model to support multiple registered credentials per user and a documented account-recovery flow before shipping this, not after.

**Example apps:** Google, Apple ID, GitHub, and most crypto exchanges (Coinbase, Kraken) now offer passkey login.

---

## 9. TOTP / MFA (Time-based One-Time Password)

**Technologies / npm packages:** `otplib`, `speakeasy`, `qrcode` (for enrollment QR codes).

**Pros:**
1. Massively reduces account takeover risk from leaked/reused passwords, at low implementation cost.
2. Works offline on the client (authenticator app) — no network dependency during code generation.
3. Well-understood, widely supported UX pattern (Google Authenticator, Authy).

**Cons:**
1. Still phishable — a fake login page can relay the OTP in real time (adversary-in-the-middle attacks).
2. Recovery codes need secure storage/design — lose the device and no backup codes means lockout.
3. Adds friction to every login unless paired with "remember this device" logic.

**Benefits:**
1. Cheap, high-ROI security layer — the single best security-per-engineering-hour investment for most apps.
2. Satisfies most compliance frameworks' "MFA required" checkbox.

**Use case scenarios:**
- *Explanatory:* Any account handling money, PII, or admin privileges.
- *Expected:* Optional-but-encouraged for regular users, **mandatory** for admin/privileged roles.

**Senior system design thought:**
MFA should be a policy layer bolted onto whatever primary auth you already have (session, JWT, OAuth) — not a separate auth system. Store the TOTP secret encrypted at rest (this is exactly where per-tenant KMS-style encryption patterns matter), and always issue one-time backup/recovery codes at enrollment.

**Example apps:** GitHub, AWS IAM, every major bank and crypto exchange.

---

## 10. Magic Links / Passwordless Email

**Technologies / npm packages:** `next-auth` (Email provider), custom implementation with signed, short-lived tokens + a transactional email service (Resend, SendGrid, Postmark).

**Pros:**
1. No password to forget, leak, or reuse — removes an entire attack surface.
2. Low-friction onboarding — great conversion for consumer signup flows.

**Cons:**
1. Fully dependent on email deliverability — spam filters, delays, or email downtime directly break login.
2. Vulnerable if the user's email account itself is compromised (single point of failure).
3. Awkward on mobile if the email app and the requesting app are different — context-switching UX friction.

**Benefits:**
1. Drastically simplifies onboarding UX for low-security-sensitivity consumer products.
2. Removes password-related support tickets entirely.

**Use case scenarios:**
- *Explanatory:* Consumer SaaS with low-to-medium security requirements, newsletter/content platforms, early-stage MVPs prioritizing signup conversion.
- *Expected:* Combined with a session or JWT issuance immediately after link verification.

**Senior system design thought:**
Magic links are a great default for early-stage products optimizing for signup conversion (this is a common choice for solo-founder / early SaaS builds) — but revisit it once you have paying B2B customers who expect password + MFA or SSO. Token TTL should be short (10–15 min) and single-use, invalidated immediately after first click.

**Example apps:** Slack (early login flow), Medium, Notion (partial), many early-stage indie SaaS products.

---

## Quick Decision Matrix

| Scenario | Recommended primary tech |
|---|---|
| Internal admin panel, need instant revocation | Session-based (Redis-backed) |
| Public API for third-party devs | API Keys (+ rate limiting) |
| Mobile app / SPA talking to your own backend | JWT (short TTL) + refresh token rotation |
| "Sign in with Google/GitHub" | OAuth2 + OIDC |
| Enterprise customer demands company SSO | OIDC (preferred) or SAML (if legacy IdP) |
| Early-stage MVP, fastest signup conversion | Magic links |
| Highest security bar for high-value accounts | Passkeys (WebAuthn) + TOTP as fallback |
| Service-to-service (microservices) | JWT with a shared/rotatable signing key or mTLS |

**Overall senior takeaway:** most production systems aren't "one auth technology" — they're a **layered stack**: OIDC/OAuth2 for identity federation and third-party login → your own short-lived JWT or session for app-level auth → API keys for programmatic/integration access → MFA/passkeys as a policy layer on top of all of it. The design skill isn't picking the "best" one — it's picking the right combination for who's authenticating (human vs. machine), how sensitive the resource is, and how much revocation control you need.
