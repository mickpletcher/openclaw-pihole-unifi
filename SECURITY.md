# Security Policy

## Supported versions

This repository is currently a pre-release implementation blueprint. No version is supported for production deployment yet.

After the first release, this table will identify the versions receiving security fixes.

| Version | Supported |
| --- | --- |
| Unreleased blueprint | No production support |

## Reporting a vulnerability

Do not open a public issue for a suspected vulnerability, exposed credential, private DNS record, client identifier, or other sensitive network information.

Use GitHub's **Report a vulnerability** option on the repository's Security tab when it is available. If private vulnerability reporting is not available, contact the repository owner privately through the contact method on the owner's GitHub profile before disclosing details.

Include only the minimum information needed to reproduce and assess the problem:

- Affected component and version or commit
- Impact and expected behavior
- Reproduction steps using sanitized values
- Any suggested mitigation

Never include live passwords, API keys, session IDs, MAC addresses, private IP addresses, internal hostnames, raw DNS queries, or production reports. Revoke or rotate an exposed credential through its issuing service; do not post it in GitHub.

The maintainer will acknowledge a complete report as availability permits, validate the finding, and coordinate remediation and disclosure. This is a volunteer-maintained project and does not provide a guaranteed response-time SLA.

## Security model

The intended implementation follows these boundaries:

- Collection and reporting are read-only by default.
- Credentials and generated network data remain local.
- Policy recommendations require human review.
- Live changes require verification and a rollback path.
- UniFi automation uses `GET` requests only.

Security reports that weaken these boundaries will be treated as high priority.
