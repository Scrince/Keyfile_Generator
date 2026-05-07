# Keyfile Generator

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Version](https://img.shields.io/badge/version-2.0.0-green.svg)
![Platform](https://img.shields.io/badge/platform-Web_Browser-orange.svg)
![Security](https://img.shields.io/badge/security-Client--Side-brightgreen.svg)

**Keyfile Generator** is a high-assurance, client-side cryptographic utility engineered for the secure generation and management of digital keys. Built on a zero-trust architecture, the application ensures that all cryptographic primitives are executed exclusively within the volatile memory of the local browser environment.

By utilizing the **Web Crypto API**, Keyfile Generator provides a seamless interface for creating high-entropy raw keys or password-protected encrypted keyfiles, bridging the gap between rigorous security standards and user-centric design.

---

## 🛡️ Security Architecture

The design philosophy is centered on **Confidentiality, Integrity, and Availability (CIA)** without the risks associated with cloud-based storage or server-side processing.

* **Zero Data Footprint:** No information is ever transmitted over the network.
* **Hardware-Ready Randomness:** Leverages `window.crypto.getRandomValues()` for non-deterministic, cryptographically secure entropy.
* **Volatile Memory Management:** Explicit zeroing of sensitive buffers (TypedArrays) to mitigate memory-scraping risks.
* **Authenticated Encryption:** Implements HMAC-CTR construction ensuring that any tampering with encrypted keyfiles is detected prior to decryption.

---

## 🚀 Core Functionalities

### 1. Key Generation & Entropy
Create 256-bit or 512-bit keys with real-time entropy visualization. Users can download raw keys for immediate use or wrap them in a password-protected layer for long-term storage.

### 2. Key Derivation (PBKDF2)
Encrypted keyfiles utilize the **PBKDF2-SHA512** derivation function with **200,000 iterations**, providing significant resistance against GPU-accelerated brute-force attacks.

### 3. Integrated Settings & UX
* **Persistent Theming:** Toggle between Light and Dark modes via a dedicated settings modal.
* **Operational Clarity:** Visual match indicators for passwords and entropy strength meters.
* **Session Reset:** A single-click global reset to clear all input fields, file buffers, and status messages.

---

## 🛠️ Usage Guide

1.  **Deployment:** Open the `Keyfile_Generator.html` file in any modern, standards-compliant browser.
2.  **Configuration:** Define your output filename and desired key length.
3.  **Security Layer:** Input a strong password if generating an encrypted keyfile.
4.  **Verification:** Use the **Decrypt** tab to import `.enc.key` files and verify key integrity.
5.  **Sanitization:** Click **Reset** at any time to purge the current session data from the UI and memory.

---

## 📜 License

This project is licensed under the MIT License - see the [SECURITY.md](SECURITY.md) for further implementation details.
