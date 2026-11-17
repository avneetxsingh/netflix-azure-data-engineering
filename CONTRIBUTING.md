# Contributing

Thanks for helping improve this project. Keep changes focused and describe how they affect the data flow.

## Workflow

1. Create a branch from the latest `main`.
2. Make one coherent change.
3. Update documentation when setup, parameters, paths, or behavior change.
4. Review the diff, commit, push, and open a pull request.

While this repository is private, contributors need explicit repository access.

## Validation expectations

- For documentation changes, check relative links, file paths, code examples, and Mermaid diagrams.
- For ADF changes, validate in a development factory and inspect affected activities.
- For notebook changes, run affected stages in an authorized development environment and check output schemas, counts, and quality expectations.
- State exactly what ran and what remains unverified. Historical notebook outputs are not evidence of a new run.

Keep notebook output compact and remove credentials, tokens, personal identifiers, and large displayed datasets before committing. Do not commit data files, Delta directories, checkpoints, or local authentication files.

Preserve attribution and third-party terms described in [NOTICE.md](NOTICE.md). Do not add a dependency or CI badge unless it represents something the project actually uses.

## Reporting an issue

Include the affected file/stage, reproduction steps, expected result, actual result, and a sanitized error. Use [SECURITY.md](SECURITY.md) for sensitive findings.
