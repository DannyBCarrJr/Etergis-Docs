# Security Architecture

## Encryption

| Layer | Algorithm | Details |
|-------|-----------|---------|
| Secret content | AES-256-GCM | Client-side, 12-byte random nonce, AAD-bound to secret_id |
| Key splitting | Shamir Secret Sharing | GF(256). Implemented in client and server, but NOT on the production path: the create flow requires a delivery passphrase per recipient, so the passphrase branch always runs and the Shamir output is discarded. The server rejects any persisted config whose shares cannot meet its threshold, so a revived path fails loudly rather than losing data |
| Per-recipient wrapping | X25519 + HKDF-SHA256 + AES-GCM | Ephemeral sender key (forward secrecy), domain-separated info/salt |
| Post-quantum wrapping | ML-KEM-768, hybrid with X25519 | Envelope v3. Hybrid by construction: an attacker must break both halves |
| Passphrase key derivation | Argon2id | m=64 MiB, t=3, p=1, 32-byte output, per-envelope random 32-byte salt |
| Envelope format | Versioned (v2, v3) | v3 carries the hybrid ML-KEM wrap. v2 (X25519-only) stays readable permanently, since a dead-man's-switch product cannot force re-enrollment |
| Password storage | Argon2id | Per-user salt, server-side |
| Data in transit | TLS 1.3 | Enforced via Cloudflare + Render |

## Design Principles

1. **Zero-knowledge by default.** The server stores only ciphertext. Decryption keys never leave the client.
2. **AAD on everything.** Every AES-GCM operation binds context (secret_id, recipient_id, domain string) into the authentication tag. Swap attacks at the DB layer are cryptographically impossible.
3. **Forward secrecy per wrap.** Each recipient key-wrap uses an ephemeral X25519 sender keypair. Compromise of the recipient's long-term key doesn't retroactively expose previously-wrapped content keys.
4. **Self-describing envelopes.** KDF parameters travel with the ciphertext. Future parameter upgrades don't break old envelopes.
5. **Domain separation everywhere.** HKDF info strings and AAD constants are unique per use-case. Reusing a derived key across contexts is structurally prevented.

## Zero-Knowledge Scope

**We cannot access:** Secret content, decryption keys, vault passphrases, plaintext attachments.

**We can access (metadata):** Email addresses, recipient emails, check-in timestamps, encrypted blob sizes, subscription status, authentication logs.

## Authentication

- Short-lived JWT access tokens (60 min) with HS256 signing (algorithm hardcoded, so no alg:none confusion)
- Refresh token rotation (single-use, invalidated on reissue)
- Google Sign-In and Apple Sign-In via JWKS local verification (no third-party auth proxy)
- Rate limiting: Cloudflare WAF edge rules + in-app per-IP middleware
- Leaked credential blocking via Cloudflare WAF
- NIST SP 800-63B password policy (12-char minimum, zxcvbn scoring, HIBP k-anonymity breach check)
- Token version column: password reset invalidates all sessions

## Access Control (RBAC)

Role hierarchy: `user` → `admin` → `owner`

- **user**: default role, standard access
- **admin**: user management, billing overview, server health
- **owner**: can grant internal plans, promote/demote admins, full authority
- **First owner:** Bootstrapped at startup via `BOOTSTRAP_ADMIN_EMAIL`. Idempotent and audited
- **Subsequent admins:** Promoted via admin console. Owner-only and audited
- **No env-based fallback.** Role state lives in the DB only.

## Infrastructure Security

- RFC 9116 security.txt (PGP clear-signed, Ed25519)
- MTA-STS enforced on email domain
- Cloudflare WAF with rate limiting, bot management, and leaked credential blocking
- CSP, HSTS, X-Frame-Options, X-Content-Type-Options headers enforced
- CI: Trivy vulnerability scanning, Python test suite (42+ tests)
- Secrets stored in GitHub encrypted secrets and Render env vars (never in code)
- Docker images built with non-root user

## Threat Model

**What we defend against:**
- Server compromise (DB dump): attacker gets ciphertext + encrypted envelopes, must brute-force Argon2id to get KEK
- Network interception: TLS 1.3 end-to-end, HSTS preload
- DB-layer swap attacks: AAD binding prevents moving wrapped keys between secrets/recipients
- Credential stuffing: Cloudflare WAF rate limits + leaked credential blocking
- Offline brute-force of vault passphrase: Argon2id m=64 MiB makes GPU/ASIC attacks economically prohibitive

**What we do NOT defend against:**
- Compromised client device (if your phone is rooted and owned, all bets are off)
- User choosing a weak vault passphrase (mitigated by strength meter + HIBP, but not prevented)
- Nation-state adversary with physical access to your device

## Vulnerability Disclosure

If you discover a security vulnerability:

1. **Email:** [security@etergis.com](mailto:security@etergis.com)
2. **PGP:** Available at [etergis.com/.well-known/security.txt](https://etergis.com/.well-known/security.txt)
3. **Full policy:** [Vulnerability Disclosure Policy](https://etergis.com/security/vulnerability-disclosure-policy.html)
4. **Acknowledgment:** Within 48 hours
5. **Critical patches:** Within 24 hours
6. **Safe harbor:** We do not pursue legal action against good-faith security researchers

We credit responsible disclosures in our [Hall of Fame](https://etergis.com/security/hall-of-fame.html).

## Key Rotation Policy

Rotate immediately if:
- Repository or infrastructure is compromised
- Secrets are accidentally committed to version control
- Team member with access departs
- Signing keys are exposed

---

Copyright 2025-2026 Carr Digital LLC. Licensed CC BY-ND 4.0; see LICENSE.
