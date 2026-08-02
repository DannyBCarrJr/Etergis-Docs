# Etergis: Secure Secret Orchestration with Dead‑Man's‑Switch Controls

> **Document version:** 2026-08-02. Describes the implementation shipped as of
> 1.1.14+113 on web, API, and Android. iOS is live at 1.1.13+111 with 1.1.14 in
> App Store review.
>
> Public technical reference for security reviewers and prospective users.
>
> **Accuracy policy:** this document describes only what ships today. Anything
> not implemented is labeled *(roadmap)*. Earlier revisions of this whitepaper
> drifted into aspirational claims (threshold cryptography as the primary
> mechanism, Ed25519-signed audit journals, a declarative policy engine,
> certificate pinning) that did not match the running system. Those claims have
> been removed or relabeled. If you are reviewing for assurance, treat any
> unlabeled statement below as a verifiable claim about production.

## Executive Summary

Etergis is a security-first orchestration layer for high-value secrets that
must remain sealed until precise conditions are met. It combines client-side
encryption, per-recipient key wrapping, and a dead-man's-switch heartbeat to
prevent unauthorized disclosure and ensure timely access when required. The
server is zero-knowledge: it holds ciphertext and metadata, and has no
capability to read user content. The owner's own copy of every secret's key is
wrapped with a hybrid post-quantum construction, so the long-lived at-rest copy
resists harvest-now-decrypt-later attacks.

Threshold cryptography and a declarative policy engine are not shipped. See
Roadmap.

**Target users:** security engineers, incident response leads, legal /
compliance teams, executives with high-value credentials, and estates /
business continuity planners.

---

## Problem

Organizations and individuals hold secrets that are both mission-critical and
catastrophic if leaked. Traditional vaults centralize risk and depend on
ongoing operator diligence. In emergencies, estates, or coercion scenarios,
secrets either fail to surface or leak uncontrolled. A better approach
requires:

- Assurance that secrets are encrypted before leaving the origin device.
- Release only when policy and time conditions are satisfied.
- A living heartbeat that detects silence and escalates appropriately.
- Custody that is provable and auditable without exposing plaintext.

---

## Solution Overview

Etergis enforces end-to-end protection with three pillars:

1. **Client-side encryption**
   - Secrets are encrypted locally before transport.
   - No plaintext touches servers or logs.
   - Keys are derived from a vault passphrase the server never receives.

2. **Per-recipient key wrapping**
   - The Data Encryption Key (DEK) protecting a secret is wrapped separately
     for each recipient. Two wrap methods ship today:
     - **Passphrase delivery** (recipients without an account): the DEK is
       wrapped under an Argon2id-derived key from a delivery passphrase the
       owner shares out of band. This is the path the current create UI uses
       for every recipient.
     - **Registered recipients** (holding a published X25519 public key): the
       DEK is wrapped using X25519 ECDH + HKDF-SHA256 + AES-256-GCM with an
       ephemeral sender key. The code path exists and is exercised for the
       owner's own copy; for delivery recipients it is not reachable from the
       shipping UI (see "Envelope versions" below).
   - The owner's own copy of the DEK is additionally wrapped hybrid
     post-quantum (X25519 + ML-KEM-768) once the owner has an ML-KEM keypair.
   - *(Roadmap)* Threshold cryptography, splitting the DEK into k-of-n shares
     with Shamir Secret Sharing so a quorum of custodians rather than every
     recipient can reconstruct, is **not on the production path**. The
     primitive and a k==n split exist in the codebase and are unit-tested, but
     the shipping create flow never reaches them.

3. **Dead-man's-switch release**
   - Heartbeats (a manual check-in tap, an authenticated API call, or a
     one-time emailed confirmation link) maintain "hold" state.
   - Missed check-ins begin a time-based escalation ladder: reminders at the
     owner's configured offsets, then release after the deadline plus a grace
     window.
   - Family-plan owners can also arm an Emergency Keyword ring, which starts a
     delivery immediately when the owner texts a private keyword from their
     verified phone.
   - Every transition is timestamped and written to an append-only audit log.
     *(Roadmap)* Cryptographic signing of journal entries.

---

## Architecture

### Components

- **Client App:** performs local encryption, DEK generation, and per-recipient
  key wrapping. Today: Flutter web, iOS, and Android, all live. QR export,
  air-gapped custodian handoff, and a CLI are *(roadmap)*.
- **Orchestration Service:** tracks check-in state, recipients, and encrypted
  blobs. Never receives plaintext secrets, delivery passphrases, vault
  passphrases, or DEKs. FastAPI + PostgreSQL.
- **Delivery worker:** a dedicated sweep worker evaluates check-in deadlines on
  a roughly 30-second tick, independently of the web tier, so a deploy or
  restart cannot pause time-based delivery.
- **Recipients:** people who receive a wrapped key, either a passphrase-delivery
  recipient or an account holder with an X25519 public key. *(Roadmap)*
  Dedicated custodian roles (HSM slots, sealed envelopes, QR codes) alongside
  threshold sharing.
- **Policy store:** *(partial)* a `Policy` record can be created and listed, but
  declarative rule **enforcement**, meaning approver roles, time delays, and
  multi-party quorum, is **not implemented**. Do not rely on it for access
  control today.
- **Audit Journal:** append-only log of authentication, administrative, and
  delivery state changes. Entries are timestamped but **not** individually
  signed. *(Roadmap)* Signed entries and a signed export endpoint.

### Cryptographic Baseline (actual implementation)

Every algorithm and parameter below is verifiable in the client and server
source at 1.1.14+113.

#### Content encryption
- **Algorithm:** AES-256-GCM
- **Nonce:** Random 12 bytes per encryption operation
- **AAD (Additional Authenticated Data):** A version-tagged domain string
  bound into every ciphertext. Domain examples: `etergis.aad.recipient-wrap.v2`,
  `etergis.aad.recipient-wrap.v3`, `etergis.aad.secret-content.v2|<secret_id>`,
  `etergis.aad.passphrase-wrap.v2|<recipient_id>|<secret_id>`. AAD prevents
  swap attacks at the storage layer.
- **Tag:** 128-bit AES-GCM authentication tag

#### Key hierarchy
- **DEK (Data Encryption Key):** Random 256-bit AES key, generated per secret
- **KEK (Key Encryption Key):** One of:
  - Derived from an X25519 ECDH shared secret via HKDF-SHA256, for classical
    recipient and owner wraps;
  - Derived from a combined X25519 + ML-KEM-768 shared secret via HKDF-SHA256,
    for hybrid owner wraps (see "Post-quantum hybrid protection");
  - Derived from a passphrase via Argon2id, for private-key envelope encryption
    and for passphrase-based recipient delivery.

#### Public-key operations
- **Classical:** X25519 (Curve25519 ECDH)
- **Post-quantum:** ML-KEM-768 (FIPS 203), used only in hybrid with X25519,
  never alone. Wire sizes are pinned and asserted at the application boundary:
  encapsulation key 1184 bytes, decapsulation key 2400 bytes, ciphertext 1088
  bytes, shared secret 32 bytes.
- **Forward secrecy:** Ephemeral sender keypair per wrap operation. The sender's
  long-term private key is not used for DEK wrapping; only the recipient's
  long-term public key is referenced.

#### Key derivation
- **From shared secret (X25519 ECDH):** HKDF-SHA256
  - Salt: Fixed domain string `etergis.salt.v2`
  - Info: Per-use-case label (e.g. `etergis.kek.recipient-wrap.v2`)
  - Output: 32 bytes (AES-256 key)
- **From combined shared secrets (hybrid):** HKDF-SHA256, detailed under
  "Post-quantum hybrid protection" below.
- **From passphrase (Argon2id):**
  - Variant: Argon2id (RFC 9106 v=0x13)
  - Memory cost: 65536 KiB (64 MiB)
  - Time cost: 3 iterations
  - Parallelism: 1 lane
  - Output: 32 bytes
  - Salt: 32 random bytes per envelope (never reused)
  - Parameters are stored inside the envelope JSON; future parameter upgrades
    do not invalidate existing envelopes. Parameters are bounds-checked on
    read, so a tampered envelope cannot declare extreme values to force memory
    exhaustion.

#### Envelope versions

"v2" and "v3" appear in two different places and mean different things. The
distinction matters when reading the schema or the code:

| Construct | Version | What it protects | Key derivation |
|---|---|---|---|
| Private-key backup envelope | EnvelopeV2 | The owner's X25519 seed, and separately the owner's ML-KEM-768 decapsulation key | Argon2id from the vault passphrase |
| DEK wrap, classical | `wrap_version` absent (v2) | A recipient's or the owner's copy of a DEK | HKDF-SHA256 over an X25519 shared secret |
| DEK wrap, hybrid | `wrap_version = 3` | The owner's copy of a DEK | HKDF-SHA256 over X25519 ‖ ML-KEM-768 shared secrets |
| DEK wrap, passphrase | `kdf_version = 2` | A delivery recipient's copy of a DEK | Argon2id from the delivery passphrase |

Both private-key backup envelopes are Argon2id EnvelopeV2 records. The
post-quantum upgrade changed how the **DEK** is wrapped for the owner, not how
the owner's private keys are backed up.

#### Envelope format (private-key backup)
- Versioned JSON, base64-wrapped for transport
- Fields: `v`, `kdf`, `kdf_params`, `salt_b64`, `ciphertext_b64`, `nonce_b64`,
  `tag_b64`, `aad`
- Current version: 2 (Argon2id)
- v1 (PBKDF2-HMAC-SHA256, 600k iterations) envelopes were used in v0.1 and are
  NOT readable by v0.4+. v0.1 to v0.4 is a clean break; existing users
  re-enrolled their vaults at upgrade.

#### Post-quantum hybrid protection (owner copy)

- **Scope:** the **owner's own copy** of every secret's DEK is wrapped with a
  hybrid X25519 + ML-KEM-768 construction. This is the harvest-now-decrypt-later
  surface: secrets sit at rest for years while the check-in timer runs.
  **Recipient delivery wraps are not hybrid.** The accurate claim is that stored
  information (the owner's copy) is hybrid post-quantum and delivery is not. Do
  not read this document as claiming end-to-end post-quantum.
- **Construction:**
  `kek = HKDF-SHA256(ikm = ss_x25519 ‖ ss_mlkem, salt = ephemeral_x25519_pub ‖ mlkem_ct[:32], info = "etergis.kek.recipient-wrap.v3")`.
  The salt binds the KEK to the specific ML-KEM ciphertext, so substituting a
  ciphertext fails the AES-GCM tag instead of silently succeeding. An attacker
  must break both X25519 and ML-KEM-768 to recover the DEK. A defect in the
  post-quantum half degrades security to classical X25519 and never below the
  pre-migration baseline.
- **Why delivery is excluded:** the passphrase delivery wrap's quantum exposure
  is Grover-only, quadratic and blunted by the Argon2id work factor, not the
  Shor break that threatens X25519. It was not a candidate for this migration on
  the merits, independent of effort. Extending hybrid wrapping to delivery would
  require durable recipient key custody, which means recipient accounts, and a
  recipient who loses a key is a failed delivery. That is the one unacceptable
  outcome for this product.
- **Server guard:** the API rejects keyed hybrid wraps on the delivery path at
  create time. The anonymous release portal only supports passphrase unwrap, so
  a recipient is never handed a ciphertext they cannot decrypt.
- **No forced re-enrollment, permanently:** new secrets wrap hybrid
  automatically once the owner has an ML-KEM keypair. A background sweep
  opportunistically re-wraps an owner's older classical wraps on a later
  passphrase-gated action, and never blocks. Records without a `wrap_version`
  stay valid forever. A dead-man's-switch product cannot require
  re-enrollment, because the owner may not be there to do it.
- **Bootstrap:** owners who predate the migration have no ML-KEM keypair. The
  sweep generates one and **reuses their existing X25519 keypair**. Minting a
  new X25519 key would orphan every previously wrapped secret.
- **Safety properties:** every re-wrap is self-verified before it is persisted,
  meaning the fresh envelope is decapsulated back to the DEK and byte-compared
  to the original, so an owner-wrap the owner could not reopen is never
  written. The server enforces an optimistic no-downgrade guard on the upgrade
  endpoint, so a stale client cannot clobber a wrap another device already
  upgraded. The DEK itself never changes, and recipient wraps and content
  ciphertext are never touched.

#### Threshold cryptography *(implemented, not on the production path)*
- **Status:** not in use. Every secret today is protected by per-recipient DEK
  wrapping. A Shamir Secret Sharing implementation over GF(256) exists in both
  client and server and is unit-tested, and the create path contains a k==n
  split, but the shipping UI requires a delivery passphrase for every recipient,
  so the passphrase branch always runs and the Shamir output is discarded.
- **Server backstop:** the server rejects any create whose wrapped shares cannot
  meet the stated threshold, so a revived flow fails loudly instead of
  persisting an unrecoverable secret.
- **Planned:** a configurable k-of-n threshold (k < n) for custodian-based
  release without every recipient needing to be online.

#### Integrity & authentication
- **Signatures:** the audit journal is append-only and timestamped but **not**
  cryptographically signed. Ed25519 signing of policy updates, custody
  acknowledgements, release decisions, and journal entries is *(roadmap)*. The
  RFC 9116 `security.txt` is PGP clear-signed with Ed25519; that is
  infrastructure signing, unrelated to user data.
- **Transport:** TLS 1.3, terminated at Cloudflare. Certificate pinning in the
  client is *(roadmap)*.

#### Second factor (account login)
- **TOTP MFA (shipped):** RFC 6238 time-based one-time passwords, with
  single-use, SHA-256-hashed backup codes. Login with MFA enabled issues a
  short-lived scoped challenge token and requires the second factor before a
  session is granted.
- **WebAuthn / hardware keys:** *(roadmap)*.

#### Login passwords (server-stored, NOT user data encryption)
- **Algorithm:** Argon2id (server-side)
- **Pepper:** Optional, configured via environment variable
- **Policy:** NIST SP 800-63B modern approach: 12-char minimum, 128-char
  maximum, no composition rules, zxcvbn entropy scoring, HIBP breach check
  via k-anonymity API.
- **Anti-enumeration:** login performs equivalent Argon2 work for unknown
  accounts, so response timing does not reveal whether an account exists.

---

## Operational Workflows

### 1) Onboarding

1. User registers with email + password (NIST 800-63B policy enforced).
2. ToS / Privacy acceptance recorded with timestamp + version.
3. User sets a separate **vault passphrase** (also under NIST 800-63B policy).
4. Client generates an X25519 keypair locally, and an ML-KEM-768 keypair
   alongside it.
5. Both public keys uploaded to server (used to wrap secrets FOR this user).
   The ML-KEM encapsulation key is optional: a client that has not uploaded one
   is wrapped X25519-only, and this is never forced.
6. Each private seed is encrypted with a vault-passphrase-derived KEK into its
   own Argon2id EnvelopeV2: one for the X25519 seed, one for the 2400-byte
   ML-KEM decapsulation key.
7. Both envelopes uploaded to the server's `UserKeyBackup` table for
   cross-device recovery, in parallel columns. Holding both is what enables
   hybrid DEK wrapping; the server cannot decrypt either without the user's
   passphrase. A client that re-uploads only its X25519 envelope does not clear
   a stored ML-KEM envelope, so a v2-only client cannot silently downgrade the
   user.
8. *(Roadmap)* Device binding via WebAuthn.

### 2) Secret ingestion

1. Client commits to a `secret_id` before any encryption. That ID is bound into
   the AAD of every AES-GCM operation for the secret, and the server persists it
   verbatim.
2. Random DEK generated locally.
3. Secret is encrypted: `ciphertext = AES-256-GCM(DEK, plaintext, AAD=secret_id)`.
   Attachments are encrypted with the same DEK.
4. For each recipient, the DEK is wrapped under the delivery passphrase
   (Argon2id) that the owner sets and shares out of band. A wrap failure aborts
   creation; an unwrapped key is never uploaded.
5. The DEK is wrapped once more for the owner, hybrid when the owner has an
   ML-KEM key on file and classical X25519 otherwise. This is what lets the
   owner reopen their own secret.
6. Orchestration service receives ONLY: metadata, recipient list, ciphertext,
   wrapped keys. Never plaintext, never DEKs, never passphrases.

### 3) Heartbeat & monitoring

1. The owner checks in: a manual tap in the app, an authenticated API call, or
   a one-time emailed confirmation link. The link opens an interstitial and
   confirms on an explicit action, so link prefetchers cannot check in on the
   owner's behalf.
2. A missed deadline begins the escalation timers; reminders are sent at the
   owner's configured offsets.
3. After the deadline plus a grace window, the switch triggers release.

### 4) Emergency Keyword Trigger (Family plan)

An alternative to waiting out the countdown, for situations where opening the
app is not safe.

1. The owner verifies a phone number, arms a ring, and chooses a private
   keyword. The keyword is stored as an HMAC-SHA256 digest keyed with a
   server-side secret and bound to the user ID, never as plaintext.
2. Texting the keyword from the verified number starts that ring's delivery.
   Inbound webhooks are signature-validated and fail closed when configuration
   is missing, replays are rejected by message-ID idempotency, and comparison is
   constant-time. Rate limiting and per-number lockout run before any keyword
   comparison.
3. The trigger is silent to the sender by design. The owner is notified by push
   and email, and an authenticated cancel window runs before delivery proceeds.
4. Non-matching inbound messages are dropped and logged. Carrier-reserved words
   are refused at keyword selection.

Residual risks accepted and documented: SMS content is visible to carriers in
transit (the keyword is single-purpose and revocable, not a reused credential),
and SIM-swap plus keyword knowledge defeats sender binding, which is equivalent
to phone custody and is covered by the cancel window.

### 5) Release

1. Release is triggered by the dead-man deadline expiring, evaluated by the
   sweep worker, or by an armed keyword ring firing. There is no interactive
   approval step today.
2. Each recipient receives a one-time release link and fetches the wrapped key
   for their identity.
3. The recipient derives the unwrapping key from the delivery passphrase
   (Argon2id) in their browser and unwraps the DEK entirely on their own device.
   The passphrase never leaves the device, and the server never receives it or
   the plaintext. Repeated failed attempts lock the release.
4. The recipient decrypts the ciphertext locally and views the content.
   *(Roadmap)* View-once, sealed-export, and ephemeral-session delivery modes.

### 6) Post-release & audit

1. The audit journal records delivery provenance: who, what, when, and which
   wrapped keys were used.
2. Recipient replacement and secret deletion are supported; the server enforces
   that added recipients have decryptable key material.
3. *(Roadmap)* Signed export for SOC and compliance archives.

---

## Threat Model

### Goals

- Prevent unilateral disclosure by any single party, including the service
  operator (zero-knowledge server).
- Protect the long-lived at-rest copy against harvest-now-decrypt-later, by
  wrapping the owner's DEK copy hybrid post-quantum.
- Resist credential theft with optional TOTP two-factor authentication and
  immediate, version-based session revocation.
- Maintain a complete append-only record of state changes.
- *(Roadmap)* Recoverability when some recipients are unavailable (threshold
  sharing), tamper evidence via signed journals, and out-of-band multi-party
  approvals. None of these ship today.

### Adversaries

- External attackers targeting clients, servers, or recipients.
- Malicious insiders or coerced recipients.
- A future adversary with a cryptographically relevant quantum computer,
  recording ciphertext today to decrypt later. Addressed for the owner's at-rest
  copy; the delivery wrap's exposure is Grover-only and blunted by Argon2id.
- Ransomware or wiper events.
- **Server compromise** (DB exfiltration). Specifically guarded against by:
  - All key material at rest in the server DB is encrypted; the server lacks
    the keys to decrypt.
  - User private-key envelopes are protected by Argon2id (m=64 MiB, t=3, p=1),
    which makes offline brute-force economically prohibitive even with GPU
    farms.
  - Per-envelope random salts prevent precomputed rainbow attacks.
  - Emergency keywords are stored as keyed HMAC digests, not plaintext.

### Mitigations

- Client-side encryption and per-recipient key wrapping, with forward secrecy
  on X25519 wraps via an ephemeral sender key.
- Hybrid X25519 + ML-KEM-768 wrapping of the owner's DEK copy.
- Optional TOTP two-factor authentication; WebAuthn is roadmap.
- Argon2id for any passphrase-derived key, configured above the OWASP minimum,
  with KDF parameters bounds-checked on read.
- Single-use rotating refresh tokens with reuse detection (a replayed token
  revokes the whole session family) and version-based revocation enforced on
  every request.
- Layered rate limiting: in-memory per-IP plus a durable database counter that
  survives restarts and spans instances, behind Cloudflare WAF at the edge.
- Append-only audit trail (unsigned today; signing is roadmap).
- *(Roadmap)* Quorum requirements and time-delayed release via the policy
  engine, and offline recovery with QR-encoded printed shares once threshold
  sharing ships.

### Out of scope

- A compromised client device. If the endpoint is owned, so is the plaintext.
- A weak vault passphrase. Strength metering and HIBP checks reduce this; they
  do not prevent it.
- Loss of both the vault passphrase and every device. By design there is no
  operator recovery path, because one would break the zero-knowledge property.

---

## Governance & Compliance

- **Data minimization:** the service never stores plaintext, passphrases, or
  DEKs.
- **Key lifecycle:** recipient replacement and secret deletion are supported;
  the server enforces that added recipients have decryptable key material.
- **Auditability:** append-only logs of auth, admin, and delivery actions.
  *(Roadmap)* Cryptographically signed logs and a signed export for external
  audit.
- **Privacy:** no advertising identifiers and no location tracking. Recipient
  names and emails and secret titles are stored as metadata; the customer-facing
  Privacy Policy states the exact metadata boundary.
- **Regulatory mapping:** supports controls relevant to SOC 2, ISO 27001,
  NIST CSF, and incident response audit requirements. Final alignment requires
  formal assessment.

---

## Implementation Notes

- **Client platforms:** Flutter web (app.etergis.com), iOS (App Store), and
  Android (Google Play), all live. Web, API, and Android are on 1.1.14+113; iOS
  is live at 1.1.13+111 with 1.1.14 in review. A CLI is *(roadmap)*.
- **Payments:** Stripe on web, RevenueCat in-app purchases on iOS and Android.
- **Rate limiting:** application-level per-IP sliding-window middleware for
  paths the Cloudflare edge cannot express, backed by a durable PostgreSQL
  counter table, with Cloudflare as the first network layer.
- **Delivery worker:** a dedicated sweep worker evaluates check-in deadlines on
  a roughly 30-second tick, independently of the web tier.
- **Wrap versions in production:** the owner's copy of a DEK is hybrid
  (`wrap_version = 3`, X25519 + ML-KEM-768) for owners who have an ML-KEM
  keypair, and classical X25519 for those who do not. Delivery wraps are
  Argon2id passphrase wraps. Classical records stay readable permanently.
- **Interoperability:** QR share encoding is *(roadmap)*, tied to threshold
  sharing.
- **Availability:** single-region managed hosting (Render) with a zero-secrets
  architecture. *(Roadmap)* Multi-region and emergency pause with quorum.

---

## Roadmap

| Version | Status | Milestone |
|---------|--------|-----------|
| v0.1.0 | shipped Apr 2026 | Recipient-based key wrap (X25519+HKDF+AES-GCM), heartbeat monitor, Flutter web client, FastAPI/PostgreSQL backend, append-only audit log |
| v0.4.0 | shipped May 2026 | Crypto modernization (PBKDF2 to Argon2id, AAD on AES-GCM, HKDF domain separation, EnvelopeV2), auth UX overhaul (Google/Apple SSO), NIST 800-63B password policy, rate limiting |
| v0.5.x | shipped Jun 2026 | Production launch, Stripe live billing, iOS + Android submissions, owner role + RBAC hierarchy, push notifications, RFC 9116 security.txt, admin console |
| v1.0 to v1.1 | shipped Jun to Jul 2026 | All platforms live (App Store + Google Play + web), payments live (Stripe web, RevenueCat IAP), TOTP MFA, passphrase-based recipient delivery with in-browser decrypt, dedicated delivery sweep worker |
| (hardening) | shipped Jul 2026 | Security-audit remediation: server-side release decryption removed (full zero-knowledge restored), refresh-token reuse detection, durable rate-limit backstop, KDF parameter bounds, check-in link anti-prefetch fix. See Appendix B |
| (PQC) | shipped Jul 2026 | Envelope v3 hybrid X25519 + ML-KEM-768 owner wrap, ML-KEM secret-key backup slot, lazy re-wrap with self-verification and server-side no-downgrade guard |
| v1.1.13 | shipped Jul 2026 | Days-protected ring, widget overhaul, recipient loop touchpoints |
| v1.1.14 | shipped Jul 2026 | Emergency Keyword Trigger live (Family plan) |
| Next | planned | Signed audit export, CLI MVP, account deletion polish |
| Later | planned | Policy engine (time locks, multi-approver quorum), Shamir threshold sharing (k-of-n) + QR-encoded shares, WebAuthn / hardware keys, certificate pinning, multi-region |
| Future | planned | Formal external crypto review, SOC 2 Type II audit, HSM-backed attestations, enterprise SSO, recipient-side keyed hybrid delivery |

---

## Appendix A: Release Policy Examples *(illustrative roadmap scenarios)*

> These examples illustrate the **target** capability of the planned policy
> engine and threshold sharing. They are **not** supported today. The current
> product releases every secret to each of its recipients when the dead-man
> deadline elapses or an armed keyword ring fires. The quorum thresholds,
> approver roles, and time delays below all depend on roadmap features.

- **Estate release:** No heartbeat for 21 days, 2-of-3 family recipients plus
  1 legal approver, 48-hour public-notice delay.
- **Incident response:** Immediate release with 3-of-5 SOC recipients and 1
  executive approver, no delay.
- **Coercion-safe:** Dual-channel challenge; if a panic code is used, require
  extra quorum and notify legal counsel.

## Appendix B: Changelog of cryptographic-relevant changes

- **2026-07-27 (v1.1.14):** Emergency Keyword Trigger live. Keywords stored as
  user-bound HMAC-SHA256 digests keyed with a server-side secret, never
  plaintext. Inbound webhooks are signature-validated and fail closed, replays
  rejected by message-ID idempotency, comparison constant-time, with per-number
  lockout and rate limiting ahead of any comparison.
- **2026-07 (Envelope v3, Phases 1 to 3):** the owner's copy of every secret's
  DEK now wraps hybrid X25519 + ML-KEM-768 (FIPS 203). New secrets wrap hybrid
  automatically once the owner has an ML-KEM keypair; existing secrets
  opportunistically re-wrap on a later passphrase-gated action, self-verified
  before persist, with no forced re-enrollment. The owner's ML-KEM
  decapsulation key gained its own Argon2id EnvelopeV2 backup slot alongside the
  X25519 seed. The server enforces an optimistic no-downgrade guard on the
  owner-wrap upgrade endpoint. Recipient delivery wraps are unchanged and remain
  Argon2id passphrase wraps; keyed hybrid delivery wraps are rejected at create
  time. This is a deliberate scope decision, not a gap.
- **2026-07 (v1.1.x security audit hardening):** a full-stack audit drove a
  batch of fixes. Cryptography and zero-knowledge relevant items:
  - **Server-side release decryption removed, restoring full zero-knowledge.**
    The delivery flow previously sent the recipient's passphrase to the server,
    which derived the key and returned plaintext. That path is gone; the release
    portal now performs the Argon2id unwrap and decryption entirely in the
    browser.
  - **Refresh-token reuse detection and version revocation on every request.** A
    replayed refresh token revokes the entire session family.
  - **KDF parameter bounds on envelope read**, so a tampered envelope cannot
    declare extreme Argon2 parameters to force memory exhaustion.
  - **Constant-work login** to remove an account-existence timing oracle.
  - **Durable rate-limit backstop** (PostgreSQL counter) behind the in-memory
    limiter, surviving restarts and spanning instances.
  - **Check-in link anti-prefetch fix**, so the emailed link no longer mutates
    state on GET.
  - **Server rejects unsatisfiable Shamir configs**, so a future threshold flow
    cannot persist an unrecoverable secret.
- **2026-06 (v1.0 to v1.1):** all platforms shipped (iOS, Android, web), TOTP
  MFA, payments (Stripe + RevenueCat), dedicated delivery sweep worker.
- **2026-06-07 (v0.5.2):** Owner role added above admin. Push notifications via
  Firebase (no cryptographic impact: notification metadata only, no secret
  content transmitted via push). RFC 9116 security.txt PGP clear-signed with
  Ed25519.
- **2026-06-05 (v0.5.0):** Production launch. No cryptographic changes from
  v0.4.0. Rate limiting fully restored on all auth endpoints.
- **2026-05-20 (v0.4.0):** Major crypto modernization. PBKDF2 to Argon2id for
  KEK derivation. AAD added to all AES-GCM operations. HKDF domain-separation
  salt + info. Versioned envelope format (EnvelopeV2). NIST 800-63B password
  policy. Clean break: v0.1 envelopes are not readable.
- **2026-05-16 (v0.3.3):** Closed Apple Sign-In aud/iss bypass (account-takeover
  vector). Replaced Google tokeninfo debug endpoint with JWKS local
  verification. Prevented silent account linking by email. JWT algorithm
  hardcoded to HS256 (closes alg:none confusion). Auth-endpoint rate limiting
  at the Cloudflare edge.
- **2026-04 (v0.1.0):** Initial public version. Recipient-based key wrapping,
  client-side encryption, dead-man's switch heartbeat, FastAPI backend.
- **Note on Shamir Secret Sharing:** multiple prior revisions of this document
  described SSS as a shipping feature. It has never protected a production
  secret. Every secret is protected by per-recipient DEK wrapping. The
  implementation exists and is unit-tested but is unreachable from the shipping
  UI; SSS remains on the roadmap.

---

Copyright 2025-2026 Carr Digital LLC. Licensed CC BY-ND 4.0; see LICENSE.
