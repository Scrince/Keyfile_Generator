# Security Policy

## Security boundary and supported use

Keyfile Generator is a static, single-file HTML application. Its security properties are those of the exact HTML and JavaScript a user opens, the browser executing it, and the endpoint on which that browser runs. The application has no backend, account system, network protocol, remote key service, or server-side recovery path. “Client-side” describes where the code executes; it does not establish that a downloaded copy is authentic, that the host is trustworthy, or that secrets are inaccessible to the browser, operating system, extensions, or user.

The keyfile envelope version is `v: 1`; it is independent of any application or release version. Security fixes should be applied to the canonical HTML and distributed copies should be replaced with a verified copy of the updated source. Do not treat a keyfile format version as an application version.

## Security-relevant design

| Function | Current implementation |
|---|---|
| Entropy source | `crypto.getRandomValues()` for key bytes, 16-byte salts, and 16-byte nonces |
| Password KDF | PBKDF2-HMAC-SHA-512 through Web Crypto; accepted counts are exactly 200,000 and 1,000,000 |
| Derived material | 64 bytes; first 32-byte slice used as the encryption PRF key, final 32-byte slice as the MAC key |
| Keystream | HMAC-SHA-512 over `nonce || byteOffset32be`, one 64-byte block per byte offset (`0`, `64`); XOR with plaintext |
| Authentication | HMAC-SHA-512 over `salt || nonce || ciphertext`; checked before decryption |
| Key payload | 32 or 64 random bytes, represented in JSON as canonical base64 |
| Input file limit | 16,384 bytes before JSON parsing |
| Dependencies | No external JavaScript libraries, imports, or runtime services in the application source |

This is the project’s interoperable `KEYFILE-ENC` v1 construction, not a standardized Web Crypto AEAD mode. It must be treated as bespoke cryptography: format compatibility is not a security proof. The two keys are adjacent slices of one PBKDF2 output, not outputs of separately labeled derivations. Passwords are UTF-8 encoded with `TextEncoder` and are not explicitly Unicode-normalized; interoperation requires identical input-to-byte behavior. Do not silently change byte layout, counter encoding, key split, MAC coverage, password encoding, or iteration semantics; such changes can break consumers and require cryptographic review and explicit format/version handling. New formats should prefer a well-reviewed AEAD construction available in Web Crypto (for example AES-GCM) and a reviewed, memory-hard password KDF where portability and distribution permit it. Argon2id is not exposed by the pure Web Crypto API.

The implementation validates required encrypted-envelope fields, magic/version, KDF identifier and iteration allowlist, exact salt/nonce/MAC lengths, canonical base64, and 32/64-byte ciphertext length. It rejects files over 16 KiB; it does not reject every unknown/additional JSON property. The MAC comparison visits each tag byte, but JavaScript and browser engines make no constant-time guarantee. JavaScript buffer overwrites are best effort and cannot erase strings, engine copies, garbage-collected objects, browser internals, swap, crash dumps, or downloaded files. In the current encryption path, cleanup of derived-key buffers and generated plaintext key bytes is not protected by `finally`; an exception before the success cleanup can leave them in memory until reclaimed. This is a limitation of the current implementation, not a guarantee of erasure on success.

## Reporting a vulnerability

Please do not open a public issue for an unpatched vulnerability and do not include real passwords, keyfiles, vaults, or other secrets in a report. Use GitHub’s private vulnerability reporting for this repository if it is enabled. If private reporting is unavailable, contact the repository maintainer through a non-public channel before publishing details. Include:

- affected release or commit and browser/operating-system versions;
- a concise impact statement and attacker prerequisites;
- reproducible steps using synthetic keys and passwords;
- a minimal proof of concept that does not expose real user material;
- any suggested mitigation, if known.

Please allow the maintainer a reasonable opportunity to investigate, prepare a fix, and coordinate disclosure. Do not access, alter, or exfiltrate data belonging to other users while validating a report. There is no guaranteed response-time or bounty commitment.

## User handling guidance

- Obtain the HTML from the project’s canonical release/source channel. For releases that provide detached signatures, verify the signature against the independently trusted project release public key; a key downloaded from the same untrusted location as the file does not independently establish authenticity.
- For high-value key generation, use a trusted, up-to-date browser on a trusted, malware-free device; close unnecessary extensions and prefer offline operation. An offline workflow reduces remote delivery and observation opportunities but does not protect against a compromised host or malicious HTML.
- Use a unique, high-entropy keyfile password. Prefer 1,000,000 PBKDF2 iterations for new keyfiles when consumer compatibility and local performance allow it. The 200,000 preset is retained for compatibility. The embedded count is attacker-controlled file input but constrained to these two values before KDF execution.
- Keep raw `.key` files and decrypted output confidential. The decrypt view displays the recovered keyfile JSON in the page; both generation and decryption passwords remain in their input fields until cleared or reset. Reset or close the page after use. Reset clears selected fields and visible output, not every browser or operating-system copy.
- Keep the password separate from the encrypted keyfile and maintain independent backups. Losing either the password or the only copy of the file can make the key unrecoverable.
- Never commit real keyfiles, passwords, screenshots, crash dumps, or diagnostic captures containing secrets.

## Release integrity

The tracked GitHub repository does not currently define a release-signing or checksum-verification workflow. Do not describe a downloaded HTML file as signed or checksum-verified unless the specific distribution actually includes those artifacts and the expected signing-key fingerprint is authenticated independently. A future signing workflow should keep private signing keys out of source control and use controlled key custody; a checksum downloaded from the same unauthenticated channel as the application is not an independent authenticity check.
