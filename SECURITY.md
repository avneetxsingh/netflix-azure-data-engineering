# Security

## Reporting a concern

Do not paste credentials, tokens, storage keys, SAS URLs, or private connection details into issues, pull requests, or notebook outputs.

Use GitHub's private vulnerability reporting feature if it is enabled for this repository. Otherwise, contact the maintainer through an existing private channel. A public issue may request a private contact method, but should not include exploit details or sensitive information.

## Working with this project

- Use managed identities and governed storage access where appropriate.
- Keep secrets in supported secret stores rather than source files.
- Reconfigure imported ADF authentication in the destination factory. Exported encrypted connection metadata is not portable authentication.
- Review notebook outputs before sharing a repository.
- Keep storage data, checkpoints, cloud credentials, and local environment settings out of Git.
- If a usable secret is exposed, revoke or rotate it at its issuing service. Deleting its latest file does not remove it from Git history.

This is a learning project without a formal security support commitment or production hardening guarantee.
