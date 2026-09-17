# Validation

## Validation Levels

### Basic

Run for documentation, governance, and small internal changes:

```powershell
pwsh -NoProfile -File ./scripts/docs-check.ps1 -FailOnGap
git diff --check
```

Both commands must exit `0`. The documentation check must report no `MISSING` or `REVIEW` rows. `git diff --check` produces no output when whitespace checks pass.

### Integration

After shell scripts are committed, run from a POSIX shell:

```sh
find . -type f -name '*.sh' -not -path './.git/*' -exec sh -n {} +
find . -type f -name '*.sh' -not -path './.git/*' -exec shellcheck --shell=sh {} +
```

Both commands must exit `0`. Run component-specific tests added with the implementation. Integration with live Pi-hole, UniFi, email, or scheduling requires sanitized local configuration and explicit authorization for any state-changing step.

### Full

Run Basic and Integration validation, then exercise the applicable manual checks in the README validation checklist. Verify read-only collection before any mutation workflow. Any Pi-hole mutation needs a dry run where supported, API readback, behavior verification, and the documented targeted rollback.

## Environment Requirements

- PowerShell 7 for `scripts/docs-check.ps1`.
- Git for repository and whitespace checks.
- A POSIX shell for planned scripts.
- ShellCheck for full shell validation after scripts exist.
- Pi-hole, UniFi, SMTP, and scheduler access only for the applicable integration test.

## Validation Matrix

| Change type | Class | Basic | Integration | Security review | Smoke test |
|---|---|---|---|---|---|
| Documentation only | 1 | Yes | No | No | No |
| Internal refactor or tests | 2 | Yes | As needed | No | Yes |
| Bug fix or feature | 3 | Yes | Yes | As needed | Yes |
| Interface, dependency, or configuration | 3 | Yes | Yes | As needed | Yes |
| Architecture, security, or deployment | 4 | Yes | Yes | Yes | Yes |

## Known Validation Limitations

- No operational scripts or configuration files are committed. Runtime, integration, and smoke validation cannot run yet. Risk: the blueprint may contain unproven commands or assumptions. Follow-up: implement each component with automated tests and validate it in a disposable or canary environment before production use.
- Markdown linting and link checking are not configured. Risk: formatting or link regressions may reach the default branch. Follow-up: add repository-pinned documentation checks when the first implementation toolchain is selected.
