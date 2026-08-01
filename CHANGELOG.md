# Changelog

## [0.5.2] - 2026-06-07: Security Compliance, Owner Role, Push Notifications

### Added

- RFC 9116 security.txt (PGP clear-signed, Ed25519, SHA-512)
- Vulnerability Disclosure Policy with safe harbor language
- Owner role (above admin): only owner can grant internal plans or promote admins
- Internal plan tier (unlimited, no billing, no expiry)
- Push notifications via Firebase Cloud Messaging (Android + iOS)
- Admin console: billing overview (MRR, plan distribution)
- Admin console: server health (DB connectivity, latency, API version)
- Landing page auto-deploy workflow
- robots.txt with AI crawler blocking + sitemap
- Security rotation reminders (GitHub Issues + cron workflow)

### Fixed

- CI rate limiter disabled in test environment (fixed 43 test failures)
- Billing portal missing route decorator (404)
- Billing Manage button popup blocker on web
- Admin bootstrap now promotes to owner (not just admin)

---

## [0.5.1] - 2026-06-06: Plan Consistency, Legal Hardening, SMS, UI Polish

### Added

- Spatial hub dashboard with countdown ring (replaces card stack)
- Account health score endpoint and checklist card
- Recipient test-notify and introduction emails
- SMS plan enforcement (Family tier only)
- Twilio A2P 10DLC registration
- ToS: Assumption of Risk, Export Controls, Electronic Communications, SMS disclosures
- Privacy Policy: No AI training commitment, Data Minimization principle
- Stripe webhook fix (construct_event)
- Email template full redesign (branded, light/dark)
- Platform-aware layout (web command center vs mobile field link)

### Fixed

- ToS Family plan recipients corrected (Unlimited → Up to 25)
- Billing page Free tier attachment display
- Recipient delete cascade (SecretRecipient + WrappedKey + DeadmanReleaseToken)

---

## [0.5.0] - 2026-06-05: Production Launch

### Added

- Production deployment: app.etergis.com (web), api.etergis.com (API)
- iOS App Store submission (build 49)
- Stripe live billing (Personal $4.99/mo, Family $9.99/mo)
- Landing page redesign (asymmetric hero, crypto specs, 3-tier pricing)
- Rate limiting restored on all auth endpoints (Cloudflare WAF + in-app middleware)
- Monthly/annual billing toggle with "Save 17%" badge

### Fixed

- 2-day Alembic crash loop resolved (missing migration on main)
- CORS: added app.etergis.com to allowed origins
- Google Sign-In: corrected client ID (was iOS, needed Web)
- Apple Sign-In: corrected services ID
- Stripe checkout: fixed frontend URL derivation for prod

---

## [0.4.1] - 2026-05-24: Edge Hotfix

### Fixed

- Apple Sign-In on Android: OAuth callback redirect completing correctly
- Cloudflare Email Routing covering all topical inboxes

---

## [0.4.0] - 2026-05-20: Cryptographic Overhaul & Auth UX

### Breaking

- All beta users must re-enroll their vaults. v0.1/v0.3 envelopes are not backwards compatible.

### Security

- PBKDF2 → Argon2id (m=64 MiB, t=3, p=1) for vault passphrase KEK derivation
- AAD bound to all AES-GCM operations (prevents DB-layer swap attacks)
- HKDF domain-separation salt + info on all X25519 key derivations
- Per-envelope random 32-byte salt (replaces deterministic salt)
- Versioned envelope format (EnvelopeV2): KDF params travel with the envelope
- JWT algorithm hardcoded to HS256 (closes alg:none confusion vector)
- Cloudflare WAF rate-limit rules on auth endpoints
- Leaked credential blocking via Cloudflare WAF
- In-app per-IP rate limiting on sensitive endpoints

### Auth UX

- Google Sign-In and Apple Sign-In (web + native)
- Password strength meter with HIBP breach check
- Auto-redirect to /register for first-time OAuth users

---

## [0.3.3] - 2026-05-16: Security Hardening & Audit Remediation

- Removed Apple Sign-In aud/iss bypass (account takeover vector)
- Replaced Google tokeninfo debug endpoint with JWKS local verification
- Prevented silent account linking by email (pre-auth takeover)
- Rate limiting on all auth endpoints
- Hardcoded HS256 algorithm
- Privacy Policy and Terms of Service rewrite (GDPR/CCPA/COPPA)

---

## [0.3.2] - 2026-05-14: Architecture Overhaul

- go_router migration (declarative routing)
- Riverpod state management
- Subscription plan limits (Free / Personal / Family)

---

## [0.3.0] - 2026-05-13: Production Deploy

- Secret editing (decrypt → edit → re-encrypt)
- Production infrastructure live
- Performance improvements

---

## [0.2.2] - 2026-05-11: HTTP Client & Observability

- Dio HTTP client with auto-retry
- structlog JSON logging with request-ID correlation
- Sentry Flutter SDK integration

---

## [0.1.0] - 2026-04: Initial Release

- Recipient-based key wrapping (X25519 + HKDF + AES-GCM)
- Dead-man's switch heartbeat monitor
- Flutter web client
- FastAPI + PostgreSQL backend
- Append-only audit log
