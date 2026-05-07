# Security Policy

## Overview
Keyfile Generator is designed to be a "Zero-Knowledge" tool. Because the application is a static HTML/JavaScript file, the security boundary is limited entirely to your local machine.

## Cryptographic Primitives

| Component | Implementation |
| :--- | :--- |
| **Randomness** | `window.crypto.getRandomValues()` |
| **Hash Function** | SHA-512 |
| **Key Derivation** | PBKDF2 (200,000 iterations) |
| **Encryption Mode** | HMAC-CTR (Encrypt-then-MAC) |
| **Key Lengths** | 256-bit / 512-bit |

## Threat Mitigation

### 1. Memory Scraping
In many web environments, sensitive strings can persist in memory. To combat this, Keyfile Generator utilizes `TypedArrays` and a `zeroArray()` function. Once a key is generated or a password is processed, the underlying memory is overwritten with zeros before the object is marked for garbage collection.

### 2. Brute Force Resistance
By utilizing 200,000 iterations of SHA-512, the computational cost of testing a single password is significantly increased, making offline dictionary attacks and brute-force attempts computationally expensive.

### 3. Data Integrity
Decryption will not proceed if the calculated HMAC does not match the stored MAC. This prevents "bit-flipping" attacks where an attacker might attempt to modify the ciphertext to alter the resulting key.

## Best Practices for Users
* **Verify Source:** Ensure you are running the official version of the HTML file.
* **Incognito Mode:** Running the tool in a Private/Incognito window disables most browser extensions, preventing potential malicious scripts from interacting with the DOM.
* **Offline Use:** For high-security requirements, save the HTML file and run it on a machine with the network interface disabled.
