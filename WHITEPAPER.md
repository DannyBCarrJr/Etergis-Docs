# Etergis: Secure Secret Orchestration with Dead‑Man's‑Switch Controls

> **Document version:** 2026-05-20 — reflects v0.4.0 implementation
>
> Public technical reference for security reviewers and prospective users.

## Executive Summary

Etergis is a security-first orchestration layer for high-value secrets that
must remain sealed until precise conditions are met. It combines client-side
encryption, threshold cryptography, and policy-driven release with a
dead-man's-switch heartbeat to prevent unauthorized disclosure and ensure
timely, auditable access when required. The platform emphasizes verifiable
custody, minimal trust in servers, and cryptographic accountability for every
state transition.

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
- Release only when quorum, policy, and time conditions are satisfied.
- A living heartbeat that detects silence and escalates appropriately.
- Custody that is provable and auditable without exposing plaintext.

---

## Solution Overview

Etergis enforces end-to-end protection with three pillars:

1. **Client-side encryption**
   - Secrets are encrypted locally before transport.
   - No plaintext touches servers or logs.
   - Strong key derivation from user factors and hardware-backed authentication.

2. **Recipient-based key wrapping** *(shipped in v0.1)* and **threshold
   cryptography** *(planned)*
   - The Data Encryption Key (DEK) protecting a secret is wrapped per
     recipient using X25519 ECDH + HKDF + AES-256-GCM.
   - Recipient-based key wrapping shipped in v0.1.0 and is the production
     mechanism today.
   - Shamir Secret Sharing splitting the DEK into k-of-n shares is a planned
     enhancement; see Roadmap. The whitepaper previously claimed Shamir was
     shipping; this was inaccurate and is corrected here.

3. **Dead-man's-switch release**
   - Heartbeats (manual, scheduled, or automated signals) maintain "hold"
     state.
   - Missed heartbeats trigger a policy-driven escalation ladder that can
     culminate in release if conditions are met.
   - Every transition is signed, timestamped, and journaled for compliance.

---

## Architecture

### Components

- **Client App / CLI:** Performs local encryption, share generation, QR
  export, and custodian handoff. Supports offline enrollment and air-gapped
  flows. Today: Flutter web + iOS + Android client. CLI is on the roadmap.
- **Orchestration Service:** Tracks policies, heartbeat status, and custody
  attestations. Never receives plaintext secrets or DEKs. FastAPI + PostgreSQL.
- **Custodian Endpoints:** People or systems holding shares or wrapped keys:
  e.g., personal devices, HSM slots, sealed envelopes, or QR codes in secured
  storage.
- **Policy Engine:** Declarative rules for who can request, approve, or
  receive reconstructed outputs, including time delays and multi-party
  approvals.
- **Audit Journal:** Append-only log of all events, signatures, and state
  changes. Exportable for SOC and compliance.

### Cryptographic Baseline (v0.2.0 — actual implementation)

This section was rewritten in May 2026 to match the shipped code. Previous
versions of this document made aspirational claims (e.g. Shamir Secret Sharing
as the primary mechanism, Argon2id for KEK derivation) that did not match the
v0.1.0 implementation. v0.2.0 closes the gaps and this section is now
authoritative.

#### Content encryption
- **Algorithm:** AES-256-GCM
- **Nonce:** Random 12 bytes per encryption operation
- **AAD (Additional Authenticated Data):** A version-tagged domain string
  bound into every ciphertext. Domain examples: `etergis.aad.recipient-wrap.v2`,
  `etergis.aad.secret-content.v2|<secret_id>`. AAD prevents swap attacks at
  the storage layer.
- **Tag:** 128-bit AES-GCM authentication tag

#### Key hierarchy
- **DEK (Data Encryption Key):** Random 256-bit AES key, generated per secret
- **KEK (Key Encryption Key):** Either:
  - Derived from X25519 ECDH shared secret via HKDF-SHA256 for recipient-based
    wraps; or
  - Derived from a user vault passphrase via Argon2id for private-key envelope
    encryption and offline passphrase-based recipient delivery.

#### Public-key operations
- **Algorithm:** X25519 (Curve25519 ECDH)
- **Forward secrecy:** Ephemeral sender keypair per wrap operation. The sender's
  long-term private key is not used for DEK wrapping; only the recipient's
  long-term public key is referenced.

#### Key derivation
- **From shared secret (X25519 ECDH):** HKDF-SHA256
  - Salt: Fixed domain string `etergis.salt.v2`
  - Info: Per-use-case label (e.g. `etergis.kek.recipient-wrap.v2`)
  - Output: 32 bytes (AES-256 key)
- **From passphrase (Argon2id):**
  - Variant: Argon2id (RFC 9106 v=0x13)
  - Memory cost: 65536 KiB (64 MiB)
  - Time cost: 3 iterations
  - Parallelism: 1 lane
  - Output: 32 bytes
  - Salt: 32 random bytes per envelope (never reused)
  - Parameters are stored inside the envelope JSON; future parameter upgrades
    do not invalidate existing envelopes.

#### Envelope format (private-key backup)
- Versioned JSON, base64-wrapped for transport
- Fields: `v`, `kdf`, `kdf_params`, `salt_b64`, `ciphertext_b64`, `nonce_b64`,
  `tag_b64`, `aad`
- Current version: 2 (Argon2id)
- v1 (PBKDF2-HMAC-SHA256, 600k iterations) envelopes were used in v0.1 and are
  NOT readable by v0.2+. v0.1 → v0.2 is a clean break; existing users
  re-enrolled their vaults at upgrade.

#### Threshold cryptography (planned — not in v0.1 / v0.2)
- **Status:** Recipient-based key wrapping ships today. Shamir Secret Sharing
  of the DEK is a planned enhancement, primarily to support custodian-based
  release without the recipient needing to be online at release time.
- **When shipped:** Shamir Secret Sharing over GF(256) for the DEK, k-of-n
  threshold. Share reconstruction client-side only.

#### Integrity & authentication
- **Signatures:** Ed25519 on policy updates, custody acknowledgements, release
  decisions, and audit journal entries.
- **Transport:** TLS 1.3 with modern cipher suites; certificate pinning in
  client.

#### Login passwords (server-stored, NOT user data encryption)
- **Algorithm:** Argon2id (via Python passlib, server-side)
- **Pepper:** Optional, configured via environment variable
- **Policy:** NIST SP 800-63B modern approach — 12-char minimum, 128-char
  maximum, no composition rules, zxcvbn entropy scoring, HIBP breach check
  via k-anonymity API.

---

## Operational Workflows

### 1) Onboarding

1. User registers with email + password (NIST 800-63B policy enforced).
2. ToS / Privacy acceptance recorded with timestamp + version.
3. User sets a separate **vault passphrase** (also under NIST 800-63B policy).
4. Client generates an X25519 keypair locally.
5. Public key uploaded to server (used by other users to wrap secrets FOR
   this user).
6. Private key seed encrypted with vault-passphrase-derived KEK → EnvelopeV2.
7. EnvelopeV2 uploaded to server's `UserKeyBackup` table for cross-device
   recovery. Server cannot decrypt the envelope without the user's
   passphrase.
8. Optional: device-binding via WebAuthn (planned for v0.5+).

### 2) Secret ingestion

1. Client derives or fetches the user's X25519 keypair.
2. Random DEK generated locally.
3. Secret is encrypted: `ciphertext = AES-256-GCM(DEK, plaintext, AAD=secret_id)`.
4. For each recipient, DEK is wrapped: ephemeral X25519 → ECDH → HKDF →
   AES-256-GCM with AAD.
5. Orchestration service receives ONLY: metadata, policy, recipient list,
   ciphertext, wrapped keys. Never plaintext, never DEKs.

### 3) Heartbeat & monitoring

1. Heartbeats posted on schedule (manual tap, TOTP-style code, or API signal).
2. Missed heartbeat begins escalation timers.
3. Recipients and approvers receive notifications per policy.

### 4) Release attempt

1. Requester initiates release; policy engine evaluates time locks, approvals,
   and risk posture.
2. Recipient client fetches the wrapped key for their identity.
3. Recipient unwraps using their long-term private key (which they unlock with
   their vault passphrase).
4. Recipient decrypts ciphertext locally.
5. Plaintext delivered per policy (view-once, sealed export, or ephemeral
   session).

### 5) Post-release & audit

1. Audit journal records full provenance: who, what, when, which wrapped keys
   were used, which devices.
2. Optional rotation: re-encrypt with new keys and refresh recipients.
3. Reports exported to SOC and compliance archives.

---

## Threat Model

### Goals

- Prevent unilateral disclosure by any single party, including the service
  operator.
- Maintain recoverability when one or more recipients are unavailable
  (full coverage requires Shamir SSS — planned).
- Resist phishing and device compromise through multi-factor and out-of-band
  approvals.
- Ensure tamper-evident operations through signed journals.

### Adversaries

- External attackers targeting clients, servers, or recipients.
- Malicious insiders or coerced recipients.
- Legal coercion requiring provable policy checks and jurisdiction-aware
  delays.
- Ransomware or wiper events.
- **Server compromise** (DB exfiltration). Specifically guarded against by:
  - All key material at rest in the server DB is encrypted; the server lacks
    the keys to decrypt.
  - User private-key envelopes are protected by Argon2id (m=64 MiB, t=3, p=1),
    which makes offline brute-force economically prohibitive even with GPU
    farms.
  - Per-envelope random salts prevent precomputed rainbow attacks.

### Mitigations

- Pure client-side encryption and per-recipient key wrapping with forward
  secrecy.
- Hardware-bound authentication factors (planned: WebAuthn).
- Argon2id for any passphrase-derived key, configured above OWASP "minimum"
  to "comfortable" tier (memory-hard, GPU-resistant).
- Quorum requirements and time delays (policy engine).
- Offline / air-gapped recovery paths using passphrase-based key wrapping +
  QR codes (planned: printed shares once SSS ships).
- Append-only, signed audit trail.

---

## Governance & Compliance

- **Data minimization:** Service never stores plaintext or DEKs.
- **Key lifecycle:** Rotation, revocation, and recipient replacement
  procedures are built-in.
- **Auditability:** Exportable, signed logs for forensics and external audit.
- **Privacy:** Pseudonymous identifiers for recipients; no location tracking.
- **Regulatory mapping:** Supports controls relevant to SOC 2, ISO 27001,
  NIST CSF, and incident response audit requirements. Final alignment
  requires formal assessment.

---

## Implementation Notes

- **Client platforms:** Flutter web (production), iOS and Android (TestFlight /
  Play Store internal testing). CLI is on the v0.5+ roadmap.
- **Interoperability:** QR spec for share encoding planned for v0.5+ (when SSS
  ships). Recipient-based wraps today use the EnvelopeV2 JSON format.
- **Availability:** Multi-region orchestration with zero-secrets architecture.
- **Resilience:** Heartbeat grace windows, staggered notifications, and
  emergency pause with quorum.

---

## Roadmap

| Version | Status | Milestone |
|---------|--------|-----------|
| v0.1.0 | shipped Apr 2026 | Recipient-based key wrap (X25519+HKDF+AES-GCM), heartbeat monitor, Flutter web client, FastAPI/PostgreSQL backend, append-only audit log |
| v0.4.0 | shipped May 2026 | Crypto modernization (PBKDF2→Argon2id, AAD on AES-GCM, HKDF domain separation, EnvelopeV2), auth UX overhaul (Google/Apple SSO), NIST 800-63B password policy, Cloudflare WAF rate limiting, self-hosted hash-wasm |
| v0.5.0 | planned | Policy engine: time-locks, multi-approver flows; signed audit export; CLI MVP |
| v0.6.0 | planned | Shamir Secret Sharing (DEK split), QR-encoded shares, custodian mobile app with hardware key integration |
| v1.0.0 | planned | Formal crypto review, SOC 2 Type II audit, HSM-backed service attestations, enterprise SSO, WebAuthn |

---

## Appendix A: Release Policy Examples

- **Estate release:** No heartbeat for 21 days, 2-of-3 family recipients plus
  1 legal approver, 48-hour public-notice delay.
- **Incident response:** Immediate release with 3-of-5 SOC recipients and 1
  executive approver, no delay.
- **Coercion-safe:** Dual-channel challenge; if panic code is used, require
  extra quorum and notify legal counsel.

## Appendix B: Changelog of cryptographic-relevant changes

- **2026-05-17 (v0.2.0):** Major crypto modernization. PBKDF2 → Argon2id for
  KEK derivation. AAD added to all AES-GCM operations. HKDF domain-separation
  salt + info. Versioned envelope format (EnvelopeV2). NIST 800-63B password
  policy. Resend email migration. Clean break — v0.1 envelopes are not readable.
- **2026-04 (v0.1.0):** Initial public version. Recipient-based key wrapping,
  client-side encryption, dead-man's switch heartbeat, FastAPI backend.
- **Pre-release:** Whitepaper originally described Shamir Secret Sharing and
  Argon2id as shipping features; these were architectural targets that did
  not match the v0.1.0 implementation. v0.2.0 brings KDF in line with the
  documented intent. SSS remains on the roadmap.
