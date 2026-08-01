# Etergis: public documentation

**Your legacy, secured.**

Etergis is an end-to-end encrypted digital continuity platform. Sensitive information
reaches the people who should receive it, on a schedule you control, and only when it
matters.

**Website:** [etergis.com](https://etergis.com)
**App:** [app.etergis.com](https://app.etergis.com)
**Security contact:** [security@etergis.com](mailto:security@etergis.com)
**Vulnerability disclosure:** [etergis.com/security/vulnerability-disclosure-policy.html](https://etergis.com/security/vulnerability-disclosure-policy.html)

---

## What's here

| Document | Description |
|----------|-------------|
| [SECURITY.md](SECURITY.md) | Security architecture overview |
| [WHITEPAPER.md](WHITEPAPER.md) | Technical whitepaper: cryptographic design and threat model |
| [CHANGELOG.md](CHANGELOG.md) | Public release history |

## How it works

1. **Create and encrypt.** Write messages, upload documents, set recipient passphrases.
   Everything is encrypted on your device before upload.
2. **Set your countdown.** Choose how long your dead-man's switch runs. Check in
   regularly to keep it alive.
3. **Automatic delivery.** If the countdown expires, recipients are notified with a
   secure link and passphrase-protected access.

## Cryptographic stack

| Layer | Algorithm |
|-------|-----------|
| Content encryption | AES-256-GCM with AAD binding (secret_id) |
| Key exchange | X25519 ECDH (ephemeral sender, forward secrecy) |
| Post-quantum wrap | ML-KEM-768, hybrid with X25519 (Envelope v3) |
| Key derivation | HKDF-SHA256 with domain-separated salt + info |
| Vault passphrase | Argon2id (m=64 MiB, t=3, p=1) |
| Recipient delivery | Passphrase-based (Argon2id), X25519-wrapped |
| Envelope format | Versioned. v3 wraps hybrid X25519 + ML-KEM-768; v2 (X25519-only) stays readable permanently |
| Key splitting | Shamir Secret Sharing: implemented in client and server, **not on the production path** (see below) |

Two points that are easy to get wrong, so they are stated plainly here.

**Post-quantum coverage is partial and deliberate.** The owner's own key backup is
wrapped hybrid, X25519 plus ML-KEM-768, because that is the long-lived copy sitting at
rest and therefore the real harvest-now-decrypt-later target. Recipient wraps remain
X25519-only for now. Extending the hybrid wrap to the recipient path is deferred, not
overlooked.

**Shamir Secret Sharing is implemented but not reachable.** The split, combine, and
server-side reconstructability checks all exist, but the create flow requires a delivery
passphrase for every recipient, so the passphrase branch always runs and the Shamir
output is discarded. Owner-configurable k-of-n thresholds remain a roadmap item. Do not
read this repository as claiming threshold sharing protects a secret today.

## Zero-knowledge guarantee

The server **never** sees:
- Secret content (plaintext)
- Decryption keys
- Vault passphrases
- Plaintext attachments

The server **does** see (metadata):
- Email addresses
- Encrypted blob sizes
- Check-in timestamps
- Subscription status

## Platforms

| Platform | Status |
|----------|--------|
| Web | Live ([app.etergis.com](https://app.etergis.com)) |
| Android | Live on Google Play |
| iOS | Live on the App Store |

## Status

**Version:** 1.1.14 (build 113)
**Owner:** Carr Digital LLC, Charlotte, NC
**Trademark:** Etergis (USPTO Serial No. 99813385)

---

## Security disclosure

If you discover a security vulnerability, please report it responsibly:

- Email: [security@etergis.com](mailto:security@etergis.com)
- PGP: available at [etergis.com/.well-known/security.txt](https://etergis.com/.well-known/security.txt)
- Full policy: [Vulnerability Disclosure Policy](https://etergis.com/security/vulnerability-disclosure-policy.html)
- We acknowledge within 48 hours
- Critical vulnerabilities patched within 24 hours

---

## License

Documentation in this repository is licensed
**[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/)**
(`SPDX-License-Identifier: CC-BY-ND-4.0`), full text in [LICENSE](LICENSE).

You may read, quote, cite and redistribute these documents verbatim, including
commercially, with attribution to Carr Digital LLC. You may not distribute modified
versions. The restriction exists for one reason: a modified security architecture
document that still carries the Etergis name would misrepresent what the product
actually does. Quoting for review, analysis or commentary is unaffected.

GitHub does not display a licence badge for this repository because its detection only
covers the CC variants listed on choosealicense.com, which excludes NoDerivatives. The
licence applies regardless.

Copyright 2025-2026 Carr Digital LLC.
