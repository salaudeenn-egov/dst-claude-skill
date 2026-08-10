# Security

This repository holds sanitized DIGIT HCM engineering knowledge. It must never
contain live secrets.

## Rules

- Never commit `.env`, credentials, private keys, auth/session tokens, or personal
  data (names, emails, phone numbers, beneficiary IDs). See `.gitignore`.
- Real credentials live only in the deployment `.env` / secure secret store.
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
