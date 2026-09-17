# AI Agent Repository Rules

- **Standard version:** 2.2
- **Project tier:** Tier 2

## Authority Resolution

Resolve documentation responsibilities through the Authority Mapping in `PROJECT-STANDARD.md`. Never create a document that duplicates an existing authority.

## Startup

1. Read this file.
2. Classify the change from Class 0 through Class 4.
3. For Class 1 and above, resolve the routed responsibilities in `PROJECT-STANDARD.md`.
4. For Class 2 and above, also read the Agent Handoff in `ASSESSMENT.md`.
5. Inspect the relevant repository state before editing.

Do not read every document for every task.

## Evidence Rules

- Never claim a check passed unless it ran in the current session.
- Report the exact command, exit code, and counts, including skipped tests and why.
- Do not weaken or remove a valid check to make a change pass.
- Do not carry an earlier health conclusion forward under a new date.
- Use `Not assessed` when evidence is unavailable.
- Material architecture claims need inline repository evidence.

## Waiver Rules

- Record a required check that cannot run as `WAIVED`, with the reason, risk, and follow-up.
- Environmental constraints are valid grounds. Time pressure and inconvenience are not.
- Move repeated transient waivers into `VALIDATION.md` or an accepted technical-debt entry.

## Documentation Rules

- Living authorities describe current truth. Update them when repository state makes them stale.
- Derived authorities are append-only. Accepted architecture decisions are immutable except for status and reciprocal supersession fields.
- The assessment is current truth, not a chronological diary.
- Archive fixed issues, resolved debt, and completed upgrades.
- Link to another authority instead of repeating its content.
- Do not infer contractual requirements from code or from the implementation blueprint.

## Repository Safety Rules

- Preserve unrelated tracked and untracked changes.
- Never commit credentials, tokens, private certificates, raw DNS queries, client names, MAC addresses, private IP addresses, internal hostnames, generated reports, or backups.
- Keep monitoring, collection, and report generation read-only by default.
- UniFi automation uses `GET` only unless the repository scope is deliberately changed through a reviewed Class 4 change.
- Pi-hole mutations require explicit human approval, a dry run where supported, verification, and a targeted rollback path.
- Use POSIX `sh` for implementation scripts unless an accepted architecture decision changes the runtime.

## Completion Rule

A task is complete only when the repository state, executed validation, recorded waivers, and living documentation agree.
