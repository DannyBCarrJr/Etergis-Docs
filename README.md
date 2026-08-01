# Etergis — Public Documentation

**Your legacy, secured.**

Etergis is an end-to-end encrypted dead-man's switch platform. It ensures sensitive information reaches the right people — automatically, securely, on your terms — only when it matters.

🔗 **Website:** [etergis.com](https://etergis.com)  
🔗 **App:** [app.etergis.com](https://app.etergis.com)  
🛡️ **Security contact:** [security@etergis.com](mailto:security@etergis.com)  
🛡️ **Vulnerability disclosure:** [etergis.com/security/vulnerability-disclosure-policy.html](https://etergis.com/security/vulnerability-disclosure-policy.html)

---

## What's Here

| Document | Description |
|----------|-------------|
| [SECURITY.md](SECURITY.md) | Security architecture overview |
| [WHITEPAPER.md](WHITEPAPER.md) | Technical whitepaper — cryptographic design and threat model |
| [CHANGELOG.md](CHANGELOG.md) | Public release history |

## How It Works

1. **Create & encrypt** — Write messages, upload documents, set recipient passphrases. Everything is encrypted on your device before upload.
2. **Set your countdown** — Choose how long your dead-man's switch runs. Check in regularly to keep it alive.
3. **Automatic delivery** — If the countdown expires, recipients are notified with a secure link and passphrase-protected access.

## Cryptographic Stack

| Layer | Algorithm |
|-------|-----------|
| Content encryption | AES-256-GCM with AAD binding (secret_id) |
| Key exchange | X25519 ECDH (ephemeral sender, forward secrecy) |
| Post-quantum wrap | ML-KEM-768, hybrid with X25519 (Envelope v3) |
| Key derivation | HKDF-SHA256 with domain-separated salt + info |
| Vault passphrase | Argon2id (m=64 MiB, t=3, p=1) |
| Envelope format | Versioned. v3 wraps hybrid X25519 + ML-KEM-768; v2 (X25519-only) stays readable permanently |
| Key splitting | Shamir Secret Sharing, threshold shares with server-side reconstructability checks |

## Zero-Knowledge Guarantee

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
| Web | Live (app.etergis.com) |
| iOS | App Store review |
| Android | Closed testing (Google Play) |

## Status

**Version:** 0.5.2 (closed beta)  
**Owner:** Carr Digital LLC · Charlotte, NC  
**Trademark:** Etergis™ (USPTO Serial No. 99813385)

---

## Security Disclosure

If you discover a security vulnerability, please report it responsibly:

- Email: [security@etergis.com](mailto:security@etergis.com)
- PGP: Available at [etergis.com/.well-known/security.txt](https://etergis.com/.well-known/security.txt)
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

© 2025-2026 Carr Digital LLC.
