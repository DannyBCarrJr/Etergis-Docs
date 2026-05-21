# Changelog

## [0.4.0] - 2026-05-20 — Cryptographic Overhaul & Auth UX

### Breaking

- All beta users must re-enroll their vaults. v0.1/v0.3 envelopes are not backwards compatible.

### Security

- PBKDF2 → Argon2id (m=64 MiB, t=3, p=1) for vault passphrase KEK derivation
- AAD bound to all AES-GCM operations (prevents DB-layer swap attacks)
- HKDF domain-separation salt + info on all X25519 key derivations
- Per-envelope random 32-byte salt (replaces deterministic salt)
- Versioned envelope format (EnvelopeV2) — KDF params travel with the envelope
- Vault passphrase whitespace preserved verbatim (NIST SP 800-63B)
- JWT algorithm hardcoded to HS256 (closes alg:none confusion vector)
- Cloudflare WAF rate-limit rules on auth endpoints
- Leaked credential blocking via Cloudflare WAF
- In-app per-IP rate limiting on sensitive endpoints

### Auth UX

- Google Sign-In and Apple Sign-In (web + native)
- New glass-card sign-in / register / forgot / reset pages
- Password strength meter with HIBP breach check
- Auto-redirect to /register for first-time OAuth users
- Vault setup page redesign

### Dependencies

- Added: dargon2_flutter (Argon2id via hash-wasm on web, libargon2 on native)
- Added: zxcvbn (password strength scoring)

---

## [0.3.3] - 2026-05-16 — Security Hardening & Audit Remediation

- Removed Apple Sign-In aud/iss bypass (account takeover vector)
- Replaced Google tokeninfo debug endpoint with JWKS local verification
- Prevented silent account linking by email (pre-auth takeover)
- Rate limiting on all auth endpoints
- Hardcoded HS256 algorithm
- Rotated Android upload keystore
- Scrubbed secrets from git history
- Privacy Policy and Terms of Service rewrite (GDPR/CCPA/COPPA)

---

## [0.3.2] - 2026-05-14 — Architecture Overhaul

- go_router migration (declarative routing)
- Riverpod state management
- Subscription plan limits (Free / Personal / Family)
- Playwright E2E test scaffolding

---

## [0.3.0] - 2026-05-13 — Production Deploy

- Secret editing (decrypt → edit → re-encrypt)
- Production infrastructure live (api.etergis.com, app.etergis.com)
- Performance improvements

---

## [0.2.2] - 2026-05-11 — HTTP Client & Observability

- Dio HTTP client with auto-retry
- structlog JSON logging with request-ID correlation
- Sentry Flutter SDK integration
- Personalized email notifications

---

## [0.1.0] - 2026-04 — Initial Release

- Recipient-based key wrapping (X25519 + HKDF + AES-GCM)
- Dead-man's switch heartbeat monitor
- Flutter web client
- FastAPI + PostgreSQL backend
- Append-only audit log
