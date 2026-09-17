# Project Standard

- **Standard version:** 2.2
- **Project tier:** Tier 2
- **Lifecycle mode:** Greenfield
- **Adopted:** 2026-09-16

## Tier Rationale

The repository is an implementation blueprint for a deployed operations system that integrates with Pi-hole, UniFi, and OpenClaw. Failures can affect DNS availability, network policy, credential handling, and private client data. Tier 2 is therefore the lowest defensible tier. Tier 2C does not apply because the repository contains no client agreement, contractual scope, or delivery sign-off.

## Authority Mapping

| Responsibility | Authority | Notes |
|---|---|---|
| Project overview | `README.md` | Existing authority |
| Current assessment | `ASSESSMENT.md` | |
| Architecture | `README.md` | Existing planned architecture; implementation state is explicit |
| Change history | `CHANGELOG.md` | |
| Defect tracking | `ISSUES.md` | |
| Technical debt | `TECH-DEBT.md` | |
| Deferred improvements | `FUTURE-UPGRADES.md` | |
| Validation | `VALIDATION.md` | |
| Operations | `README.md` | Existing setup, operations, recovery, and troubleshooting guide |
| Decision history | `docs/decisions/` | Derived and append-only |
| Resolved history | `docs/archive/` | Derived and append-only |
| Development rules | `PROJECT-STANDARD.md` | This file |
| Agent rules | `AGENTS.md` | |
| Agreed scope | Not required at this tier | Tier 2C only |
| Agreed design | Not required at this tier | Tier 2C only |
| Scope amendments | Not required at this tier | Tier 2C only |
| Requirement traceability | Not required at this tier | Tier 2C only |

This table and `.docs-authority.json` must agree. One file may serve more than one responsibility when it already contains the authoritative content. Do not create a competing document.

## Document Classes

- Living documents describe current truth and are corrected when repository state changes.
- Derived documents preserve history. Append entries and do not rewrite accepted records.
- Governance documents define the repository process and are versioned when the method changes.

Tier 2 has no contractual authorities. Do not infer contractual requirements from the blueprint or implementation.

## Source of Truth

For what the system currently does, executable repository state, configuration, and tests take precedence over documentation. For planned behavior, the README is a non-contractual blueprint. A mismatch between the blueprint and a future implementation is reported in the assessment and resolved deliberately.

## Change Classes

| Class | Name | Examples |
|---|---|---|
| 0 | Trivial | Formatting, whitespace, typo fixes, ignore-list additions |
| 1 | Documentation | Human-facing documentation only |
| 2 | Internal implementation | Refactors and test additions without interface changes |
| 3 | Functional | Features, fixes, dependencies, configuration, interfaces |
| 4 | Architectural or security | Trust boundaries, deployment, authentication, authorization, data model |

Class 0 skips the documentation lifecycle. Class 1 and above require a routed documentation review. Class 4 changes presume an architecture decision record. If no record is needed, the completion report must say why.

## Routed Review

| Change type | Read before changing | Review after changing |
|---|---|---|
| Documentation only | Target authority | Target authority |
| Internal refactor | Architecture, technical debt, validation | Change history, technical debt, assessment |
| Test-only | Validation | Change history, validation |
| Bug fix | Assessment, validation, defect tracking | Change history, defect tracking, assessment |
| New feature | Overview, architecture, validation, deferred improvements | Overview, change history, architecture, assessment |
| Interface or API | Overview, architecture, validation | Overview, change history, architecture, validation, assessment |
| Dependency | Architecture, technical debt, validation | Change history, architecture, technical debt, assessment |
| Configuration | Operations, architecture, validation | Change history, operations, assessment |
| Architecture | Architecture, decision history, validation | Change history, architecture, decision history, assessment |
| Security | Architecture, validation, operations | Change history, architecture, validation, operations, assessment |
| Deployment | Operations, architecture, validation | Change history, operations, architecture, assessment |

Resolve responsibility names through the Authority Mapping. Read the assessment Agent Handoff on every Class 2 or higher task.

## Evidence and Waivers

Material living-document claims must cite a repository path, command result, or configuration. Use `Not assessed` when evidence is unavailable.

Report validation as `PASS`, `FAIL`, `BLOCKED`, `WAIVED`, or `SKIPPED`. A required command that cannot run needs the unmet requirement, reason, risk, and follow-up. Environmental constraints are valid grounds. Time pressure is not.

## Documentation Lifecycle

1. Classify the change.
2. Resolve and read the routed authorities.
3. Inspect the relevant repository state.
4. Make the smallest coherent change.
5. Run the validation required by `VALIDATION.md`.
6. Record validation evidence and waivers in the completion report.
7. Update only authorities made stale by the result.
8. Run `scripts/docs-check.ps1 -FailOnGap`.

Living authorities are rewritten, not appended as diaries. Fixed issues, resolved debt, and completed upgrades move to `docs/archive/`. Accepted architecture decisions are immutable except for status and supersession fields.

## Security Rules

- Never commit credentials, tokens, private certificates, raw DNS queries, client identifiers, private addresses, internal hostnames, generated reports, or backups.
- Use placeholders in examples.
- Keep collection and reporting read-only by default.
- Do not weaken a security boundary to simplify implementation.
- Every live mutation requires human review, verification, and a targeted rollback path.

## Standard Maintenance

Material changes to this method require a version update here, a review of `AGENTS.md`, and an entry in `CHANGELOG.md`. Run the quarterly review defined by Standard 2.2: execute docs-check, review standing waivers, reconcile repository state, archive resolved entries, and remove living documentation that no longer earns its maintenance cost.
