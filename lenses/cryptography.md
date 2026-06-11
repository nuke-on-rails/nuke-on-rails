# Lens: Cryptographic Failures

OWASP 2025 A04. How the app protects secrets at rest and signs/encrypts what it trusts. Brakeman barely touches this; the failures are in *how* crypto is used, which only reading the code reveals. Look in `lib/`, `app/models/`, and anywhere `OpenSSL`, `Digest`, `encrypt`, `sign`, or a `*_token` appears.

Reference: Rails Security Guide and OWASP A04 — link findings to them.

## The encryption-oracle pattern (RailsGoat's planted flaw, and it's common)

The highest-value finding in this lens: **one encryption/signing routine reused for both a trust token and user-supplied data.** If the same `encrypt_sensitive_value` that mints the auth/remember-me cookie also encrypts a value the user can submit and read back (a bank account number rendered in a JSON response), the user has an encryption oracle — they feed in any user id, get back a valid auth token, and authenticate as anyone. Trace every use of a crypto helper: if one call signs/encrypts an authorization value and another encrypts attacker-controllable data with the same key, that is a confirmed-critical finding.

## What to flag

- **Hand-rolled crypto instead of Rails primitives.** Custom `OpenSSL::Cipher` routines where `ActiveRecord::Encryption` (encrypted columns), `ActiveSupport::MessageEncryptor`/`MessageVerifier` (signed/encrypted tokens), or `has_secure_password` (bcrypt) is the right tool. Rails 8 ships all three — rolling your own is the smell.
- **Static or reused IV.** A fixed `iv` (or no IV) means identical plaintext yields identical ciphertext — pattern leakage. IVs must be random per encryption.
- **Weak or fast hashing for passwords.** MD5/SHA1/SHA256 for password storage (RailsGoat stores an MD5-looking digest); passwords need bcrypt/scrypt/argon2 via `has_secure_password`. Fast hashes are for integrity, not passwords.
- **Hardcoded keys/IVs** in `lib/` or initializers instead of Rails credentials / ENV (overlaps `lenses/secrets.md`).
- **Token comparison with `==`** instead of `ActiveSupport::SecurityUtils.secure_compare` — timing attack (overlaps `lenses/authentication.md`).
- **Sensitive columns stored in plaintext** — bank/tax/health/document numbers without `encrypts :column` (ActiveRecord Encryption). Read the schema for obviously sensitive column names with no encryption.
- **Reversible encryption where a one-way hash belongs** — if a value is only ever compared, not displayed, it should be hashed, not decryptable.

## Severity and remedies

An encryption oracle or a recoverable auth-token scheme is confirmed-critical — state the takeover path in one sentence. Plaintext sensitive columns and weak password hashing rank just below, depending on data class. Remedies name the Rails primitive: `encrypts` for columns, `MessageEncryptor`/`MessageVerifier` for tokens, `has_secure_password` for passwords, random IVs, `secure_compare` for comparisons. For field/file encryption beyond ActiveRecord Encryption, the Lockbox gem (and KMS Encrypted for key management) is the named remedy. Never recommend a hand-rolled fix where a Rails primitive or established gem exists.
