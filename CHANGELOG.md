# Changelog

Public release history. Entries are grouped by milestone, so a single heading may
cover several store builds.

## [1.1.20] - 2026-08-19: Session Resilience and Unlock Recovery

### Fixed

- Signing in on a second device no longer signs the first one out. Refresh tokens moved from one slot per account to a session per signed-in device, so ordinary multi-device use no longer looks like token theft
- A cold start that cannot reach the API no longer signs the user out. Session restore now distinguishes a server-rejected token from an unreachable server, and only the server's own rejection clears stored credentials
- Vault unlock recovers from keyboard-mangled passphrases and stale cached envelopes: on failure it retries sensible variants (a trimmed passphrase, the trailing-space artifact of swipe keyboards) and falls back to a fresh server copy, healing the local cache on success. Every rung failing reports a plain "this passphrase doesn't match this vault" instead of a cryptic error
- Web unlock no longer refuses a valid key backup after a mobile client re-posts it

### Security

- A replayed refresh token now revokes only the session it fired in, which makes reuse detection a real theft signal instead of a side effect of owning two devices. A just-rotated token keeps a single-use 60-second grace so a client killed mid-rotation recovers; a global token version remains the kill switch for password changes and logout-all
- Logout signs out the calling device only, and takes effect immediately
- Vault passphrase rotation is all-or-nothing: the new server copy is read back and proven to open on the device before the local cache is touched, so a partial rotation fails loudly at the moment it happens
- Passphrase fields disable autocorrect and suggestions, and a new passphrase with leading or trailing whitespace draws a visible warning (warned, never trimmed: a deliberate space stays legal)

---

## [1.1.19] - 2026-08-16: Trust Surfaces

### Added

- Lifeline settings state the worst-case delivery time as one number: check-in window plus grace period, computed live as you adjust either
- The dashboard protection score now includes "Test that a contact can be reached," and arming the Lifeline offers to send a test. The check proves reachability, not decryption, and the copy says so
- The create flow asks whether each contact can produce their delivery passphrase without you, and the Lifeline manifest reports unconfirmed passphrases as their own category
- The release portal tells a recipient without the passphrase what their options are, plainly including that neither Etergis nor support can reset or unlock it
- Crash reporting is operational on all three platforms, with personal data scrubbed before anything is sent

### Changed

- A fresh account's Lifeline defaults now match the product's own advice: 7-day check-in window and 1-hour grace period. Existing accounts are untouched

### Fixed

- A rejected contact create no longer looks like nothing happened: duplicate emails and phone numbers are flagged while typing, and a server refusal reopens the dialog with the reason shown inside it

---

## [1.1.18] - 2026-08-15: Dependency Majors

### Changed

- Framework majors taken deliberately on both tiers (starlette 1.x on the API, go_router 17 in the client), each verified against the specific behavior changes those majors introduced, plus a batch of minor updates. Soaked on dev before promotion

---

## [1.1.16 to 1.1.17] - 2026-08-11 to 08-12: iOS Push Fixed

### Fixed

- iOS push notifications had never worked: from the App Store launch until this release, no iOS device had ever registered a push token. Four stacked defects, from a missing push entitlement in the build to a broken token handoff, every one of them silent. All four are closed, and the fix is confirmed by real device registrations in production rather than inferred from a green build
- Push registration failures now report at every stage instead of failing silently, which is the more important half: the entitlement fixes one bug, and the reporting is what makes the next one findable

### Changed

- A release build can no longer reach the App Store unless the signed binary carries the production push entitlement. A sandbox-signed store build registers tokens that never receive anything, which reads as success from every angle except the one that matters

---

## [1.1.15] - 2026-08-09: Delivery History, Backend Audit Closed

### Added

- A contact's page now shows delivery history: what was sent, when, and what happened to it. Before this, a test delivery reported "sent" in a vanishing message and left no record, so the only way to ask "did that work?" was to send another one

### Security

- Outbound notifications are rate-capped: 3 per hour to any one contact and 50 per day per account, and sender display names are length-capped and sanitized before they reach the subject line of mail to people who never signed up for anything
- The delivery retry queue re-checks current state before sending: cancelling a countdown settles the queue at that moment, and removing a recipient withdraws the release link already issued to them
- A withdrawn release link can no longer be brought back through the expired-link extension flow. Recipients see "no longer available" instead of being offered an extension nobody can grant
- Social sign-in only attaches to an account that has verified its own email address, closing a path where pre-registering someone else's address could capture their later sign-in
- Two-factor codes are spent on first use, and five wrong codes in a row start a 15-minute cooldown
- Password reset links are single-use
- Refused requests are logged with enough context to tell a real caller from testing, while continuing to withhold credentials and release tokens

---

## [1.1.14] - 2026-07-27: Emergency Keyword Trigger

### Added

- Emergency Keyword Trigger (Family plan): text a private keyword from a verified phone to start a delivery immediately, for moments when opening the app is not safe
- Phone number verification
- Trigger alerts deep-link straight to the rings page, where cancel lives

### Security

- Keywords are stored as user-bound HMAC-SHA256 digests keyed with a server-side secret, never as plaintext
- Inbound SMS webhooks are signature-validated and fail closed when configuration is missing; replays are rejected by message-ID idempotency
- Per-number lockout and rate limiting run before any keyword comparison, and the comparison is constant-time
- The trigger is silent to the sender by design. The owner is notified by push and email, with an authenticated cancel window before delivery proceeds
- Carrier-reserved words (STOP, HELP and similar) are refused at keyword selection

---

## [1.1.13] - 2026-07-18: Sealed Countdown Brand, Days Protected, Widget Overhaul

### Added

- New Sealed Countdown identity across every surface: app, icons, share cards, and landing page
- Days-protected counter and a reworked home screen widget
- Opt-in recipient introduction email when a contact is added
- Safety all-clear notice when a late check-in follows an alert
- Public `GET /healthz/worker` liveness endpoint, so external monitoring can page when the delivery worker stalls

### Changed

- Terminology: ring center "RELEASED" is now "Sent"; "Streak freeze" is now "Grace period"

---

## [1.1.11] - 2026-07-10: Post-Quantum Hybrid Key Wrapping (Envelope v3)

### Security

- The owner's copy of each secret's data key is now wrapped with a hybrid X25519 + ML-KEM-768 (FIPS 203) construction. An attacker must break both halves. This is the long-lived at-rest copy and therefore the real harvest-now-decrypt-later target
- The owner's ML-KEM secret key gained its own Argon2id envelope backup alongside the X25519 seed, for cross-device recovery
- Existing secrets upgrade opportunistically on a later passphrase-gated action. Every re-wrap is verified against the original key before it is persisted, so a wrap the owner could not reopen is never written
- No forced re-enrollment, permanently. Classical X25519 records stay valid and readable. A dead-man's-switch product cannot require re-enrollment, because the owner may not be there to do it
- The upgrade endpoint enforces an optimistic no-downgrade guard, so a stale client cannot clobber a wrap another device already upgraded
- Scope, stated plainly: recipient delivery remains Argon2id passphrase wrapping and is not hybrid. Etergis is not end-to-end post-quantum

### Added

- OpenAPI snapshot contract test: schema drift fails CI

---

## [1.1.10] - 2026-07-05: Home Screen Widgets

### Added

- iOS (WidgetKit) and Android home screen widgets: self-updating countdown, real ring visual, and one-tap "I'm here" check-in

---

## [1.1.8 to 1.1.9] - 2026-07-01 to 07-04: Full-Stack Security Audit Remediation

### Security

- **Server-side release decryption removed, restoring full zero-knowledge.** The release portal now performs the Argon2id unwrap and decryption entirely in the recipient's browser. The server never receives delivery passphrases or plaintext
- Refresh-token reuse detection: a replayed token revokes the entire session family. Token-version revocation is enforced on every request
- KDF parameters are bounds-checked on envelope read, so a tampered envelope cannot declare extreme values to force memory exhaustion
- Constant-work login removes an account-existence timing oracle
- Durable database-backed rate-limit backstop behind the in-memory limiter, surviving restarts and spanning instances
- The emailed check-in link no longer mutates state on GET, so link prefetchers cannot reset a deadline on the owner's behalf
- The server rejects any secret whose wrapped shares could not meet their stated threshold
- Server-side upload size limits, MFA hardening, and webhook idempotency

---

## [1.1.5 to 1.1.7] - 2026-06-25 to 07-01: All Platforms Live

### Added

- iOS and Android live on the App Store and Google Play alongside web, as of 2026-07-01
- RevenueCat in-app purchases on iOS and Android
- Biometric session unlock
- SMS contacts by phone number (Family plan)
- Mission Control dashboard

---

## [1.1.4] - 2026-06-17: In-App Review, Haptics, Accessibility

### Added

- In-app review prompts, haptic feedback, and ring accessibility improvements
- Welcome email

### Fixed

- Delivery single point of failure

---

## [1.1.0] - 2026-06-15: Rings and Lifeline

### Added

- Rings and Lifeline convergence: check-ins and delivery unified under a ring model
- Plan gating across ring types
- Encryption export compliance declaration

---

## [1.0.0 to 1.0.2] - 2026-06-10 to 06-13: Ring System and Safety Check-In

### Added

- Ring system with the Safety Check-In ring type
- Navigation and retention overhaul
- TOTP multi-factor authentication with single-use backup codes
- Admin portal
- Email template overhaul and landing polish

---

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
