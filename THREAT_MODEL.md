# Threat Model

## 1. Assets
- **Primary Asset:** The generated secret key.
- **Secondary Asset:** The user-provided encryption password.

## 2. Adversaries & Threats
- **Online Attackers:** Low risk, as the app has no network functionality.
- **Local Malware:** High risk. Malicious software on the host machine could log keystrokes (passwords) or read memory contents.
- **Brute Force:** Attackers attempting to crack an exported `.enc.key` file.
- **Tampering:** A malicious actor modifying the encrypted file to cause a MAC mismatch or attempt to leak data through error states.

## 3. Mitigations
- **High Iteration Count:** 200,000 iterations of PBKDF2 slow down offline brute-forcing of passwords.
- **MAC Verification:** The tool performs a strict equality check on the HMAC-SHA512 signature before attempting decryption to prevent padding or tampering attacks.
- **No Dependencies:** By avoiding external JS libraries, the tool minimizes the supply-chain attack surface.
