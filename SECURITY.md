# Security

This repository holds sanitized DIGIT HCM engineering knowledge. It must never
contain live secrets.

## Rules

- **No credentials in skills — ever.** Never commit `.env`, credentials, usernames,
  private keys, auth/session tokens, Basic-auth headers, or personal data (names,
  emails, phone numbers, beneficiary IDs). This applies to every file type: markdown,
  Postman collections, scripts, screenshots, and example payloads. Use `<REDACTED>` /
  `<REDACTED-USER>` or `{{variable}}` placeholders. See `.gitignore`.
- Real credentials live only in the deployment `.env` / secure secret store —
  and those stores must themselves be protected (OS keyring, encrypted vault, or
  an encrypted volume), not a plaintext file on an unencrypted disk.
- **Encrypt saved data at rest.** Any locally saved campaign data (ES/DB extracts,
  Excel/CSV exports, recovery inputs, tracer dumps) contains beneficiary PII and
  must live on an encrypted volume (e.g. BitLocker) or in password-protected
  archives, with the password shared out-of-band. Delete extracts when the task
  is done. Raw campaign data never enters this repository in any form — only
  aggregate findings and methods.
- Run a secret sweep (passwords, tokens, `Basic [A-Za-z0-9+/=]+` blobs,
  username-next-to-password, API-key formats) before every push.
- Production endpoint hosts and internal cluster URLs are redacted to
  `<REDACTED...>` placeholders on purpose — do not restore them.
- Keep this repository **private** (internal organizational knowledge).

## If you find a secret

1. Do not push further.
2. Remove the secret from the working tree and, if already pushed, purge it from
   git history (`git filter-repo` / BFG) and force-push.
3. **Rotate** the exposed credential immediately — assume it is compromised once
   committed, even if later removed.
4. Note the incident to the repository owner.
