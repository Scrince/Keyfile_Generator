# Threat Model — Keyfile Generator

**Assessment basis:** the repository’s current `Keyfile_Generator.html`, README, and release workflow. This model describes the implementation as inspected; it is not a certification, formal verification, or claim that the custom encrypted-keyfile construction has been independently cryptanalyzed. Reassess it whenever code, dependencies, delivery, file formats, or supported deployment assumptions change.

## 1. System and security objective

Keyfile Generator is a static browser application for generating 256-bit or 512-bit random key material, exporting it as raw `KEYFILE` JSON or as password-protected `KEYFILE-ENC` JSON, and validating/decrypting those formats. The HTML embeds its CSS and JavaScript. Cryptographic operations use Web Crypto; imported keyfiles are read with the browser File API; output is offered through a Blob download or displayed in a read-only textarea. The application has no server-side component, account, persistence service, telemetry, or intentional network API call in the current source.

**Primary security objective:** when run from authentic, unmodified code in a trustworthy browser and endpoint, generate key material from the browser CSPRNG and, when requested, protect its exported representation such that an offline holder of the encrypted file cannot recover the key without finding the password, subject to the construction and password entropy.

**Additional objectives:** reject unsupported or malformed envelopes; authenticate encrypted content before releasing plaintext; keep ordinary operation local to the browser; preserve the specified keyfile interoperability contract.

**Out of scope:** protecting a user from a compromised OS, browser, browser extension, hardware, malicious administrator, screen/keyboard recorder, or hostile code that is already executing in the page’s origin; secure deletion; key escrow/recovery; vault-file encryption; and guaranteeing confidentiality of a raw `.key` file after export.

## 2. Architecture and data flow

```text
User password → DOM string → UTF-8 bytes ─┐
                                          ├─ PBKDF2-HMAC-SHA-512(salt, count) → 64 bytes
Random 16-byte salt ──────────────────────┘                                  ├─ first 32: HMAC keystream key
                                                                             └─ last 32: MAC key
                                                                                   │
Random 16-byte nonce ────────────────────────────────────────────────────────────┤
Random 32/64-byte key plaintext ────────────────────────────────────────────────┤
                                                                                   ▼
                                                   HMAC-SHA-512 counter stream XOR plaintext
                                                                                   │
                                                                                   ▼
                                                   HMAC-SHA-512(macKey, salt || nonce || ciphertext)
                                                                                   │
                                                   JSON envelope → Blob download

Selected local file → size check (≤16 KiB) → JSON parse → schema/base64 validation
  ├─ raw: validate 32/64-byte payload → display JSON
  └─ encrypted: validate → derive → verify MAC → decrypt → display raw JSON
```

No server or remote key service participates. This reduces the network attack surface, but delivery remains security-critical: whoever can replace the HTML before execution can read passwords and keys, alter generated output, or falsely report successful verification. A browser loaded from a web origin also remains subject to that origin’s delivery and compromise risks. The repository does not define a hosting server or HTTP security headers, and the HTML does not declare a Content Security Policy; hosted deployments inherit those controls from the operator.

## 3. Assets and security properties

| Asset | Required property | Consequence of compromise |
|---|---|---|
| Generated raw key (32 or 64 bytes) | Confidentiality and integrity | Direct compromise of any system or vault relying on it |
| Password used to wrap a key | Confidentiality | Offline recovery of the encrypted keyfile becomes feasible according to password entropy and KDF cost |
| Encrypted keyfile | Integrity and availability; confidentiality conditional on password strength | Tampering should fail authentication; loss prevents recovery; weak password permits offline guessing |
| Source HTML and release artifacts | Authenticity and integrity | Malicious code can steal all secrets entered or generated in its page context |
| Browser process, OS, extensions, endpoint | Integrity and confidentiality | Full compromise defeats in-page protections |
| File-format metadata and interoperability | Correct interpretation | Downgrade, rejection, or incompatible consumers if fields are changed or ignored |
| User-visible decrypted output and clipboard/downloads | Confidentiality | Exposure through screen capture, clipboard managers, browser state, backups, or other local software |

## 4. Trust boundaries

1. **Release source → executing page.** Git hosting, release assets, mirrors, local copies, and the verification key establish which code runs. Detached signatures help only when the signing key fingerprint is independently trusted.
2. **User / DOM → JavaScript.** Passwords and decrypted values are JavaScript strings and DOM values. Same-page script, developer tools, extensions with sufficient access, accessibility software, and browser instrumentation may observe them.
3. **Browser cryptographic APIs → JavaScript.** Web Crypto provides CSPRNG and PBKDF2/HMAC operations, but results and input buffers cross into a garbage-collected runtime. Buffer overwrites are best effort, not guaranteed erasure.
4. **Local file → parser and crypto.** Imported data is attacker-controlled, including when supplied by a legitimate user. Size, structure, encoding, and iteration values require validation before expensive work.
5. **Page → downloaded or displayed output.** Blob downloads and textarea rendering place sensitive material under browser and endpoint control; the application cannot control subsequent copies, backups, indexing, or sharing.
6. **Application format → external consumers.** Other implementations must agree on byte concatenation, counter numbering/endianness, KDF count, key split, and MAC coverage. Compatibility assumptions are not authenticated by the outer envelope as a signed statement.

## 5. Adversaries and capabilities

- **Offline file holder:** obtains an encrypted keyfile and can perform unbounded offline password guesses. No rate limit can be imposed by this client application.
- **Malicious file supplier:** provides malformed or crafted JSON to trigger parser, resource, validation, or crypto-path failures.
- **Local observer or malware:** reads keystrokes, DOM, process memory, temporary files, downloads, clipboard, or screenshots.
- **Code-delivery attacker:** controls a release, hosting path, mirror, update channel, or signing key and can substitute executable HTML.
- **Malicious/compromised extension or same-origin script:** observes or modifies the page’s secrets and output.
- **Active file modifier:** changes ciphertext or metadata and seeks plaintext leakage, malleability, or a consumer-specific parsing discrepancy.
- **Availability attacker:** supplies oversized or pathological local files or induces expensive repeated PBKDF2 work; can also exhaust browser memory through unrelated host activity.

The design does not claim resistance to the endpoint, browser, or executing code being compromised. An attacker able to alter the running page is equivalent to an attacker able to read and replace all secrets handled by it.

## 6. Attack paths and controls

### A. Offline password guessing

**Path:** acquire `.enc.key` → guess candidate password → run PBKDF2-HMAC-SHA-512 using embedded salt and allowed `kdf.i` → derive keys → test the 64-byte HMAC.

**Controls:** 16-byte random salt; two accepted iteration counts (200,000 and 1,000,000); 64-byte PBKDF2 output split into distinct 32-byte encryption and MAC key slices. For each guess, the 64-byte tag gives the file holder a definitive offline check of the candidate password.

**Residual risk:** PBKDF2 is CPU-hard, not memory-hard. Cost is bounded and fully offline; a weak or reused password remains vulnerable. The application cannot enforce password entropy. A higher iteration count increases legitimate delay as well as attacker cost. Argon2id is unavailable in pure Web Crypto.

### B. Ciphertext alteration, truncation, or substitution

**Path:** modify `ct`, salt, nonce, or envelope fields → cause decryption or consumer disagreement.

**Controls:** validate required envelope fields and lengths; authenticate `salt || nonce || ciphertext` with HMAC-SHA-512; verify before decrypting; reject invalid MAC and unsupported KDF parameters; ciphertext length must be exactly 32 or 64 bytes.

**Residual risk:** this is a custom HMAC-counter-mode construction, not a standardized AEAD API. The outer fields `magic`, `v`, `kdf.algo`, and `kdf.i` are not literally included in the MAC input; the first three are fixed by validation, and changing the iteration count changes the derived key and should make authentication fail. Cross-implementation differences or future format drift can still create security failures. The comparison loop processes every tag byte, but JavaScript and browser engines do not guarantee constant-time execution.

### C. Malformed or adversarial imported files

**Path:** provide JSON with invalid types/encodings, large payloads, unsupported parameters, or boundary values → induce exception, excess allocation, expensive KDF, or UI confusion.

**Controls:** reject files larger than 16,384 bytes before parsing; strict canonical base64 decoding; fixed expected salt, nonce, and MAC lengths; ciphertext must decode to 32 or 64 bytes; PBKDF2 iteration count must be one of two explicit values; raw and encrypted magic/version are checked.

**Residual risk:** the size bound limits but does not eliminate parser/UI denial of service. A syntactically valid encrypted file can still force up to 1,000,000 PBKDF2 iterations per user-triggered attempt. Validation checks required properties but permits unknown/additional JSON properties. The HTML file-input `accept` attribute is only a picker hint, not a security control. Error details are rendered as text (`innerText`), which avoids interpreting attacker-provided messages as HTML in the inspected paths.

### D. Page substitution and software supply chain

**Path:** compromise source hosting or distribution → deliver altered HTML that captures password and key material or forges success.

**Controls:** self-contained page with no third-party runtime libraries; source is available in the public GitHub repository.

**Residual risk:** no runtime mechanism can verify the page that is currently executing. The tracked repository does not define a release-signing or checksum-verification workflow, so no independent artifact-integrity mechanism is currently specified. A locally opened page can still be replaced or edited; a future signed release is only as trustworthy as its key custody and fingerprint-verification process.

### E. Endpoint, browser, and memory disclosure

**Path:** keylogger, malware, extension, crash reporter, debugger, swap, or physical observer collects password, plaintext key, or file.

**Controls:** secrets are processed locally; code overwrites selected `Uint8Array` buffers; reset clears specified DOM fields and output; no app-managed persistence or telemetry is present in the inspected source.

**Residual risk:** JS strings are immutable, typed-array zeroing is best effort, Web Crypto internal key objects are not erasable by this code, and browsers/OSes may retain copies. Passwords remain in their DOM input fields until cleared or reset. In the encrypted generation path, the derived buffer and its key slices are wiped only on successful completion, and the generated plaintext key is wiped only after a successful download setup; exceptions can skip those cleanup calls. Decryption uses `finally` for its byte buffers, but cannot erase the password string, serialized plaintext output, or Web Crypto internals. Reset is UI hygiene, not cryptographic erasure. Endpoint compromise defeats the application.

### F. Randomness failure

**Path:** insecure or unavailable RNG produces predictable key, salt, or nonce.

**Controls:** `crypto.getRandomValues()` supplies all generated byte arrays; no `Math.random()` path is used for secrets in the inspected crypto implementation.

**Residual risk:** security depends on a correctly implemented browser/OS CSPRNG and execution in a secure context supported by that browser. If the API is absent or fails, generation should fail rather than substitute a weaker source.

### G. Output handling and filename confusion

**Path:** export or decrypted display causes accidental disclosure, overwrite, or misleading file handling.

**Controls:** encrypted and raw formats have distinct suffixes; generated filename is a download hint; decryption output is readonly and resettable.

**Residual risk:** filename text is user-controlled and may have platform-specific interpretation; the application does not control destination directory, overwrite behavior, backup software, or subsequent sharing. Raw output is intentionally unencrypted. The decrypted output remains visible until reset/close and may be copied by host software.

## 7. Cryptographic contract and limits

The current encrypted envelope is:

```json
{
  "magic": "KEYFILE-ENC",
  "v": 1,
  "kdf": { "algo": "PBKDF2-SHA512", "i": 1000000, "s": "<16-byte salt, base64>" },
  "n": "<16-byte nonce, base64>",
  "ct": "<32- or 64-byte ciphertext, base64>",
  "mac": "<64-byte tag, base64>"
}
```

The 64-byte PBKDF2 result is divided into adjacent 32-byte slices; it does not use separately labeled/domain-separated PBKDF2 invocations. For each 64-byte plaintext block, the stream is `HMAC-SHA-512(encKey, nonce || byteOffset32be)` and the ciphertext is XOR with that stream; the four-byte big-endian field is the byte offset (`0`, `64`, …), not a block ordinal (`0`, `1`, …). The tag is `HMAC-SHA-512(macKey, salt || nonce || ciphertext)`. Decryption authenticates before applying the same stream transform. The only permitted KDF counts are 200,000 and 1,000,000. Salt and nonce are 16 bytes each. Password strings are UTF-8 encoded by `TextEncoder` without an explicit Unicode normalization step; consumers must reproduce the same byte encoding. These details are interoperability-critical.

This repository does not provide a formal proof or independent cryptographic audit for this construction. The present 32/64-byte payload bound keeps the counter domain far from exhaustion. The scheme has no standardized algorithm identifier beyond the project envelope name and is not equivalent to AES-CTR, AES-GCM, or the `.lfv` vault format. Any redesign should introduce an explicit new format version and migration/interop plan; never reinterpret existing v1 ciphertext.

## 8. Security requirements for changes

1. Preserve fail-closed behavior: do not release decrypted bytes before successful authentication.
2. Keep all attacker-controlled size and algorithm parameters bounded before allocation or expensive cryptographic work.
3. Do not add network access, remote scripts, analytics, external fonts, or runtime dependencies without revisiting the trust model and delivery boundary.
4. Do not log or place passwords, raw keys, decrypted payloads, or full keyfile contents in status text, diagnostics, issue reports, or telemetry.
5. Handle sensitive buffers in `try/finally` paths; document that JavaScript zeroization remains best effort.
6. Treat any cipher/KDF/MAC/serialization change as a security-sensitive format change; specify test vectors and cross-consumer behavior in review.
7. Avoid claiming “zero knowledge,” “secure deletion,” “unbreakable,” or equivalent properties that the implementation and runtime cannot establish.
8. Update this model and `SECURITY.md` when attack surfaces, cryptographic behavior, delivery, or supported assumptions change.

## 9. Risk summary

| Priority | Risk | Current exposure | Practical treatment |
|---|---|---|---|
| High | Executable or endpoint compromise | Modified HTML, hostile extensions, malware, or a compromised browser can capture or forge all page data | Establish a signed release process with an independently authenticated fingerprint; use a trusted endpoint; keep runtime code self-contained |
| High | Release artifact provenance | The tracked repository does not define signed releases or checksum verification | Publish authenticated checksums/signatures and document independent fingerprint verification before treating downloads as verified |
| High | Weak keyfile password | An encrypted file supports unbounded offline guesses | Require a unique, high-entropy password; use the 1,000,000-iteration preset where compatible |
| Moderate | Custom construction assurance and interoperability | HMAC counter stream is bespoke and not independently audited here | Obtain expert cryptographic review; define test vectors and versioned migration before format changes; prefer standardized AEAD for a new format |
| Moderate | Exception-path secret retention | Current encrypted generation cleanup is skipped if an exception occurs before success cleanup | Put cleanup in `finally` blocks and retain the limitation that runtime-level erasure cannot be guaranteed |
| Low to moderate | Malformed-file resource exhaustion | Import is bounded to 16 KiB and KDF counts are allowlisted, but a valid file can still trigger 1M PBKDF2 work | Preserve size/parameter bounds and avoid automatic or repeated KDF execution |

The local-only architecture removes a backend from the key-handling path, but it does not make the browser or release channel a trusted vault. The highest-value mitigations are authenticating the executable artifact, using a strong unique password, keeping the endpoint trustworthy, establishing sound release provenance, and preserving strict format validation and authentication-before-decryption.
