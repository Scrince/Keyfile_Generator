# Changelog

All notable changes to the **Keyfile Generator** will be documented in this file. This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] - 2024-05-23

### 🛡️ Security Enhancements
- **Memory Sanitization:** Integrated manual zero-filling for `Uint8Array` buffers containing passwords and raw key material to prevent post-operation memory leaks.
- **Entropy Visualization:** Added real-time entropy bar to provide visual confirmation of cryptographic strength.
- **KDF Hardening:** Standardized on 200,000 iterations for PBKDF2-SHA512.

### ✨ User Experience
- **Settings Modal:** Introduced a dedicated popup for application settings, including a Light/Dark mode toggle.
- **Visual Stability:** Implemented fixed-height container logic to prevent UI "jumping" during tab switching or file imports.
- **Privacy Toggles:** Added "Show Password" checkboxes for both Generation and Decryption modules.
- **Global Reset:** Added a "Reset" button to both tabs to instantly clear all sensitive inputs and restore defaults.

### 🎨 UI & Design
- **Branding:** Transitioned from "Keyhole Tool" to "Keyfile Generator."
- **Refined Aesthetic:** Modernized the interface with a professional card-based layout, utilizing the 'Inter' typeface for enhanced legibility.
- **Theme Default:** Application now defaults to **Light Mode** with high-contrast Dark Mode support.

---

## [1.0.0] - 2024-05-20
### Added
- Initial release with Web Crypto API integration.
- Support for HMAC-CTR authenticated encryption.
- JSON-based keyfile export format.
