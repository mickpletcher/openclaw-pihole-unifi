# Contributing

Thank you for helping improve OpenClaw Pi-hole and UniFi Operations.

## Before you begin

This project is currently an implementation blueprint. Check the README's project-status section and existing issues before starting substantial work. Open an issue first for changes that alter the architecture, security boundary, Pi-hole object model, or UniFi access model.

## Development principles

- Keep monitoring and report generation read-only by default.
- Never commit credentials, raw DNS queries, client identifiers, private addresses, internal hostnames, generated reports, or scheduler backups.
- Use POSIX `sh` for project shell scripts unless a change explicitly documents another runtime.
- Keep email delivery opt-in.
- Require a dry run, verification, and targeted rollback for every mutation workflow.
- Preserve existing Pi-hole rules, groups, subscriptions, and client assignments unless the requested change explicitly targets them.

## Branches and commits

Use short, descriptive branch names:

- `feat/<topic>` for new capabilities
- `fix/<topic>` for corrections
- `docs/<topic>` for documentation
- `chore/<topic>` for maintenance

Write imperative commit subjects and keep unrelated changes in separate commits.

Use annotated semantic-version tags in the form `vMAJOR.MINOR.PATCH` for releases, for example `v0.1.0`. Do not create a release tag until the referenced implementation is present and the release commit passes validation.

## Pull requests

1. Fork or branch from `main`.
2. Make the smallest coherent change.
3. Update tests and documentation when behavior changes.
4. Run the validation commands below.
5. Open a pull request and complete the checklist.

For shell changes, run:

```sh
find . -type f -name '*.sh' -exec sh -n {} +
```

When ShellCheck is available, run:

```sh
find . -type f -name '*.sh' -exec shellcheck --shell=sh {} +
```

Sanitize all fixtures and examples. Use reserved placeholders such as `PIHOLE_IP_OR_HOSTNAME`, `UNIFI_CONSOLE`, and `example.com` rather than real network values.

## Reporting problems

Use the issue templates for reproducible bugs and feature requests. Follow [SECURITY.md](SECURITY.md) for vulnerabilities or sensitive findings.
