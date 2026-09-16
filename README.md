# OpenClaw Pi-hole and UniFi Operations

[![Repository validation](https://github.com/mickpletcher/openclaw-pihole-unifi/actions/workflows/ci.yml/badge.svg)](https://github.com/mickpletcher/openclaw-pihole-unifi/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Project status: blueprint](https://img.shields.io/badge/status-implementation%20blueprint-orange.svg)](#project-status)

Build a local, privacy-first operations layer that lets OpenClaw monitor Pi-hole, generate daily DNS policy reviews, correlate active clients with UniFi, and stage proposed blocking changes safely.

## Project status

**Pre-release implementation blueprint.** The operational scripts and configuration files described in this guide have not yet been committed to the repository. The current repository documents the intended architecture, security boundaries, and implementation plan; it is not yet a deployable toolkit.

Do not follow the installation commands until the referenced files are present in a tagged release. Contributions that help implement and test the planned components are welcome.

This guide is written for a new GitHub repository. It is not tied to one network, account, IP address, email address, or computer username.

The design has four rules:

1. Monitoring and reporting are read-only by default.
2. Credentials and generated network data stay local.
3. Suggested policy changes require human review.
4. Every live change needs verification and a rollback path.

## What this project does

The completed setup can:

- Check the Pi-hole web service and DNS service independently.
- Track health transitions without repeating the same alert continuously.
- Record Pi-hole summary metrics when an app password is available.
- Create a local 24-hour domain review.
- Compare allowed and blocked queries with maintained blocklists and allowlists.
- Add plain-language domain descriptions and likely blocking impact.
- Protect important service domains from broad automatic recommendations.
- Store reviewer decisions in a local feedback file.
- Collect a reduced, read-only list of currently connected UniFi clients.
- Correlate Pi-hole clients with UniFi clients by MAC address first and IP address second.
- Place untested block candidates in an isolated Pi-hole canary group.
- Export only tested and approved domains for publication.
- Optionally email the completed report.

It does not automatically change Pi-hole policy during monitoring or report generation.

## Scope

This guide assumes:

- OpenClaw is already installed on a macOS or Linux host.
- Pi-hole v6 is already running.
- UniFi integration is optional.
- The host can reach Pi-hole and, if used, the UniFi console.
- The completed repository will contain the scripts and configuration files described below.

The original implementation was operated on macOS. The shell scripts use POSIX `sh`. Linux paths and scheduling commands will differ where noted.

## Architecture

```text
Pi-hole v6 API and DNS
        |
        | read-only collection
        v
OpenClaw host                         UniFi Network API
  monitor-pihole.sh                         |
  generate-domain-review.sh <---------------+
  refresh-unifi-inventory.sh          GET requests only
        |
        v
Local private state
  ~/.openclaw/service-env/
  ~/.openclaw/workspace/main/memory/
        |
        v
Human review and canary test
        |
        v
Reviewed public blocklist or allowlist
```

The Git repository stores code, templates, and public list definitions. It must not store secrets, raw DNS queries, reports, client names, MAC addresses, private IP addresses, internal hostnames, or backups.

## Repository layout

The planned implementation uses this structure:

```text
.
├── README.md
├── monitor-pihole.sh
├── generate-domain-review.sh
├── refresh-curated-lists.sh
├── refresh-unifi-inventory.sh
├── run-daily-review.sh
├── send-domain-review-email.sh
├── setup-pihole-candidate-test.sh
├── sync-candidate-test-rules.sh
├── export-approved-denylist.sh
└── config/
    ├── list-sources.csv
    ├── protected-domains.txt
    ├── domain-descriptions.tsv
    └── recommendation-feedback.csv
```

Recommended local-only runtime layout:

```text
~/.openclaw/
├── service-env/
│   ├── pihole-app-password
│   ├── unifi-base-url
│   ├── unifi-api-key
│   └── smtp-app-password
└── workspace/main/
    ├── bin/
    │   └── installed scripts
    ├── config/
    │   └── local configuration and downloaded lists
    └── memory/
        ├── pihole-monitor-state
        ├── pihole-monitor-report.md
        ├── pihole-metrics.csv
        ├── pihole-domain-review.md
        └── unifi-clients.json
```

## Security boundary

Follow these rules before installing anything:

- Keep all credentials under `~/.openclaw/service-env`.
- Set that directory to mode `700`.
- Set credential files to mode `600` or `400`.
- Never paste a password or API key into chat, an issue, a commit, or a report.
- Never put credentials or generated inventory in OneDrive, Dropbox, iCloud Drive, or Git.
- Give OpenClaw reduced local data, not unrestricted access to the UniFi API.
- Use only UniFi `GET` requests in automated collection.
- Do not follow HTTP redirects while sending a UniFi API key.
- Require HTTPS for the UniFi base URL.
- Treat a UniFi key as privileged unless its read-only authorization has been conclusively proven.
- Do not test write denial by sending harmless-looking write requests. A `400` or `404` does not prove the key lacks write access.
- Do not automatically apply block or allow recommendations.
- Back up an existing scheduler file before editing it.
- Do not create a duplicate scheduled job if one already exists.

Add these patterns to `.gitignore`:

```gitignore
.env
*.key
*.secret
service-env/
memory/
reports/
backups/
unifi-clients.json
pihole-metrics.csv
pihole-domain-review.md
pihole-monitor-report.md
```

## Prerequisites

Install or confirm these tools on the OpenClaw host:

- `sh`
- `curl`
- `jq`
- `dig`
- `awk`
- `sed`
- `sort`
- `gzip`
- `base64` if email delivery is enabled
- `plutil` and `launchctl` on macOS

Check them:

```sh
for command in sh curl jq dig awk sed sort gzip; do
  command -v "$command" >/dev/null || printf 'Missing: %s\n' "$command"
done
```

Stop if a required command is missing. Install dependencies through the operating system's normal package manager.

## 1. Set the site-specific values

Do not hardcode personal values in public scripts. Use environment variables.

At minimum, set:

```sh
export PIHOLE_HOST="PIHOLE_IP_OR_HOSTNAME"
```

Supported Pi-hole settings should include:

| Variable | Purpose | Recommended default |
|---|---|---|
| `PIHOLE_HOST` | Pi-hole hostname or IP | No public default |
| `PIHOLE_PASSWORD_FILE` | Pi-hole v6 app-password file | `$HOME/.openclaw/service-env/pihole-app-password` |
| `PIHOLE_CONNECT_TIMEOUT` | Connection timeout in seconds | `3` |
| `PIHOLE_STATE_FILE` | Last monitor state | `$HOME/.openclaw/workspace/main/memory/pihole-monitor-state` |
| `PIHOLE_REPORT_FILE` | One-line monitor result | `$HOME/.openclaw/workspace/main/memory/pihole-monitor-report.md` |
| `PIHOLE_METRICS_FILE` | CSV metric history | `$HOME/.openclaw/workspace/main/memory/pihole-metrics.csv` |
| `PIHOLE_DOMAIN_REPORT_FILE` | Daily review report | `$HOME/.openclaw/workspace/main/memory/pihole-domain-review.md` |
| `PIHOLE_REVIEW_CONFIG_DIR` | Local review configuration | `$HOME/.openclaw/workspace/main/config` |
| `PIHOLE_LIST_DIR` | Downloaded curated lists | `$PIHOLE_REVIEW_CONFIG_DIR/lists` |
| `PIHOLE_CANDIDATE_GROUP` | Canary group name | `Candidate-Test` |
| `PIHOLE_CANDIDATE_LIST_URL` | Hosted candidate list | User-controlled HTTPS URL |

Supported UniFi settings should include:

| Variable | Purpose | Recommended default |
|---|---|---|
| `UNIFI_API_KEY_FILE` | Local API-key file | `$HOME/.openclaw/service-env/unifi-api-key` |
| `UNIFI_BASE_URL_FILE` | Local console-origin file | `$HOME/.openclaw/service-env/unifi-base-url` |
| `UNIFI_INVENTORY_FILE` | Reduced local inventory | `$HOME/.openclaw/workspace/main/memory/unifi-clients.json` |
| `UNIFI_CONNECT_TIMEOUT` | Connection timeout | `3` |
| `UNIFI_REQUEST_TIMEOUT` | Total request timeout | `30` |
| `UNIFI_PAGE_SIZE` | API page size | `200` |
| `UNIFI_SKIP_CERTIFICATE_CHECK` | Allow a trusted local self-signed certificate | `false` |

Do not publish a real IP address, username, email address, password-file content, or API key as a default.

## 2. Create protected local directories

Run on the OpenClaw host:

```sh
install -d -m 700 "$HOME/.openclaw/service-env"
install -d -m 700 "$HOME/.openclaw/workspace/main/bin"
install -d -m 700 "$HOME/.openclaw/workspace/main/config"
install -d -m 700 "$HOME/.openclaw/workspace/main/memory"
```

Confirm permissions:

```sh
ls -ld \
  "$HOME/.openclaw/service-env" \
  "$HOME/.openclaw/workspace/main/bin" \
  "$HOME/.openclaw/workspace/main/config" \
  "$HOME/.openclaw/workspace/main/memory"
```

## 3. Create a Pi-hole app password

Create a dedicated Pi-hole v6 app password in the Pi-hole administration interface. Use a credential intended for this local integration.

Enter it directly in the host terminal. The input stays hidden:

```sh
umask 077
printf 'Paste the Pi-hole app password: ' >&2
IFS= read -r -s PIHOLE_SECRET
printf '\n' >&2
printf '%s' "$PIHOLE_SECRET" > "$HOME/.openclaw/service-env/pihole-app-password"
unset PIHOLE_SECRET
chmod 600 "$HOME/.openclaw/service-env/pihole-app-password"
```

Verify metadata without printing the password:

```sh
test -s "$HOME/.openclaw/service-env/pihole-app-password"
ls -l "$HOME/.openclaw/service-env/pihole-app-password"
```

Pi-hole v6 API sessions use this flow:

1. `POST /api/auth` with the app password.
2. Read `.session.sid` from the response.
3. Send the SID in the `X-FTL-SID` header.
4. Use `DELETE /api/auth` when finished.

Every script must clear password variables after authentication and close the API session in a trap or `finally` block.

## 4. Install scripts and configuration

> This section applies after the implementation files are added. See [Project status](#project-status).

From the repository root:

```sh
for script in ./*.sh; do
  /bin/sh -n "$script"
done

install -m 700 ./*.sh "$HOME/.openclaw/workspace/main/bin/"
install -m 600 ./config/* "$HOME/.openclaw/workspace/main/config/"
```

Do not copy a hardcoded personal path into `run-daily-review.sh`. The wrapper should resolve `$HOME`:

```sh
#!/bin/sh
set -eu

BIN_DIR="${OPENCLAW_OPERATIONS_BIN:-$HOME/.openclaw/workspace/main/bin}"

"$BIN_DIR/generate-domain-review.sh"

if [ "${PIHOLE_EMAIL_REPORT:-false}" = "true" ]; then
  "$BIN_DIR/send-domain-review-email.sh"
fi
```

Email must be opt-in. A normal report validation must not send email.

## 5. Configure the review files

### `config/list-sources.csv`

List the public feeds used for comparison:

```csv
name,url
balanced,https://example.com/curated-blocklist.txt
strict,https://example.com/curated-blocklist-strict.txt
device,https://example.com/curated-blocklist-device.txt
policy,https://example.com/curated-blocklist-policy.txt
allowlist,https://example.com/curated-whitelist.txt
project,https://example.com/project-denylist.txt
```

Use stable HTTPS URLs. If a project list has not been published yet, allow an empty local bootstrap copy. Do not silently ignore failures for established feeds.

### `config/protected-domains.txt`

Add broad service suffixes that must never become automatic block suggestions:

```text
# One suffix per line
apple.com
icloud.com
microsoft.com
microsoftonline.com
windowsupdate.com
google.com
googleapis.com
```

Protection does not prove every subdomain is safe. It prevents a broad recommendation from breaking an important platform. Review exact subdomains manually.

### `config/domain-descriptions.tsv`

Use a tab-separated catalog:

```text
domain	description
example.com	Likely service purpose and the expected functional impact if blocked.
```

Resolution should use the most specific exact hostname first, then walk toward parent domains. Descriptions are context. They are not proof of ownership, behavior, or blocking safety.

Validate the catalog for:

- Duplicate keys.
- Missing descriptions.
- Invalid delimiters.
- Recommendations that resolve only to fallback text.

Unknown domains should say that client attribution and owner verification are required. Do not guess.

### `config/recommendation-feedback.csv`

Use this format:

```csv
domain,decision,note,updated_at,test_started_at,test_completed_at
# Decisions: block, allow, ignore, review
```

Decision meanings:

- `block`: Candidate for canary testing.
- `allow`: Keep functional access.
- `ignore`: Suppress a known non-actionable item.
- `review`: More evidence is required.

A `block` entry with an empty `test_completed_at` is eligible for the canary group. Only a tested block with a completed timestamp is eligible for publication.

## 6. Validate basic Pi-hole health

Run the monitor manually:

```sh
PIHOLE_HOST="PIHOLE_IP_OR_HOSTNAME" \
  "$HOME/.openclaw/workspace/main/bin/monitor-pihole.sh"
```

The monitor must check two independent services:

- The HTTPS admin endpoint returns `200`, `301`, `302`, `307`, or `308`.
- `dig` can resolve Pi-hole's built-in `pi.hole` name through the target server.

Expected state lines:

```text
PIHOLE_OK ...
PIHOLE_ALERT ...
PIHOLE_DOWN ...
PIHOLE_RECOVERED ...
```

`PIHOLE_ALERT` is the first failed sample. Repeated failures become `PIHOLE_DOWN`. The first successful sample after failure becomes `PIHOLE_RECOVERED`.

When the app-password file is present, the monitor may append:

- Total queries.
- Blocked queries.
- Blocking percentage.
- Queries per second.
- Unique domains.
- Forwarded queries.
- Cached queries.

The metrics file is private operational data. Do not commit it.

## 7. Configure optional read-only UniFi inventory

The collector uses the official local UniFi Network API base:

```text
https://UNIFI_CONSOLE/proxy/network/integration/v1
```

It sends the API key in `X-API-Key` and uses only:

```text
GET /sites
GET /sites/{siteId}/clients
```

The API responses are paginated. Process `offset`, `limit`, `count`, `totalCount`, and `data` until all currently connected clients are collected.

Create a dedicated UniFi identity with the least access your controller supports. If there is any doubt about the key's rights, treat it as privileged.

Store the console origin:

```sh
umask 077
printf '%s\n' 'https://UNIFI_CONSOLE' \
  > "$HOME/.openclaw/service-env/unifi-base-url"
chmod 600 "$HOME/.openclaw/service-env/unifi-base-url"
```

Enter the API key locally:

```sh
umask 077
printf 'Paste the UniFi API key: ' >&2
IFS= read -r -s UNIFI_SECRET
printf '\n' >&2
printf '%s' "$UNIFI_SECRET" > "$HOME/.openclaw/service-env/unifi-api-key"
unset UNIFI_SECRET
chmod 600 "$HOME/.openclaw/service-env/unifi-api-key"
```

The collector must:

- Set `umask 077`.
- Reject API-key files with modes other than `400` or `600`.
- Reject a base URL that does not start with `https://`.
- Never use `curl --location` while carrying the API key.
- Contain no `POST`, `PUT`, `PATCH`, or `DELETE` request.
- Write output atomically.
- Store only the reduced fields needed for correlation.

Recommended reduced output:

```json
{
  "generatedAt": "UTC_TIMESTAMP",
  "access": "read-only",
  "clients": [
    {
      "siteId": "LOCAL_VALUE",
      "id": "LOCAL_VALUE",
      "name": "LOCAL_VALUE",
      "type": "WIRED_OR_WIRELESS",
      "macAddress": "LOCAL_VALUE",
      "ipAddress": "LOCAL_VALUE",
      "networkId": "LOCAL_VALUE",
      "vlanId": "LOCAL_VALUE",
      "connectedAt": "LOCAL_VALUE",
      "lastSeenAt": "LOCAL_VALUE"
    }
  ]
}
```

Run the collector:

```sh
"$HOME/.openclaw/workspace/main/bin/refresh-unifi-inventory.sh"
```

For a known, trusted local console using a self-signed certificate:

```sh
UNIFI_SKIP_CERTIFICATE_CHECK=true \
  "$HOME/.openclaw/workspace/main/bin/refresh-unifi-inventory.sh"
```

Prefer installing a trusted certificate. Do not use the skip option for an untrusted or remote endpoint.

Expected result:

```text
UNIFI_INVENTORY_OK file=... clients=... generated=...
```

Validate structure without printing the inventory:

```sh
INVENTORY="$HOME/.openclaw/workspace/main/memory/unifi-clients.json"

jq -e '
  (.generatedAt | type == "string") and
  (.access == "read-only") and
  (.clients | type == "array")
' "$INVENTORY" >/dev/null

jq '{
  generatedAt,
  access,
  clientCount: (.clients | length),
  wired: ([.clients[] | select(.type == "WIRED")] | length),
  wireless: ([.clients[] | select(.type == "WIRELESS")] | length)
}' "$INVENTORY"
```

The official client endpoint guarantees currently connected clients. It is not a permanent device-history database.

## 8. Generate the daily domain review

Run directly during validation:

```sh
PIHOLE_HOST="PIHOLE_IP_OR_HOSTNAME" \
  "$HOME/.openclaw/workspace/main/bin/generate-domain-review.sh"
```

The generator should:

1. Refresh configured public lists.
2. Refresh UniFi inventory if configured.
3. Create a short-lived Pi-hole v6 session.
4. Read allowed and blocked domain activity for the review period.
5. Read current Pi-hole lists, groups, and clients.
6. Compare observed domains with maintained feeds.
7. Apply protected-domain and reviewer-feedback rules.
8. Add the best matching domain description.
9. Correlate Pi-hole and UniFi clients locally.
10. Write a Markdown report.
11. Close the Pi-hole API session.

The report should contain:

- Existing-list coverage issues.
- New evidence-based block candidates.
- Blocked domains that may need allow or unblock review.
- A confidence score from 0 through 100.
- Evidence supporting each recommendation.
- A practical domain description and expected blocking impact.
- UniFi inventory freshness and read-only status.
- Pi-hole and UniFi client-correlation results.

Confidence measures evidence strength. It does not measure whether blocking is safe.

The report must state:

```text
Policy: recommendations only. No Pi-hole rules are changed automatically.
```

Check the report without dumping private client data into logs:

```sh
REPORT="$HOME/.openclaw/workspace/main/memory/pihole-domain-review.md"
test -s "$REPORT"
grep -nE '^# Pi-hole domain review|^Policy:|^## |^Inventory status:' "$REPORT"
```

## 9. Correlate Pi-hole and UniFi clients safely

Use this order:

1. Exact normalized MAC-address match.
2. Current IP-address match.
3. No match.

Do not guess when MAC and IP evidence conflict. Flag the record for review.

MAC-first matching matters because IP addresses can change. Randomized Wi-Fi MAC addresses can still prevent stable matching. Report that limitation instead of forcing a match.

Keep all correlation local. Public output should contain counts and method descriptions, not client-level identifiers.

## 10. Create an isolated candidate-testing group

Do this only after the read-only workflow is stable.

The one-time setup should:

1. Create a Pi-hole group named `Candidate-Test` if it does not exist.
2. Subscribe only that group to a separate hosted candidate list.
3. Leave the subscription disabled while the hosted list is empty.
4. Assign no clients automatically.
5. Preserve every existing list, group, rule, and client assignment.

Preview or run the setup:

```sh
PIHOLE_HOST="PIHOLE_IP_OR_HOSTNAME" \
PIHOLE_CANDIDATE_LIST_URL="https://example.com/project-denylist.txt" \
  "$HOME/.openclaw/workspace/main/bin/setup-pihole-candidate-test.sh"
```

Enable the subscription only after the first approved domain is published:

```sh
PIHOLE_HOST="PIHOLE_IP_OR_HOSTNAME" \
PIHOLE_CANDIDATE_LIST_URL="https://example.com/project-denylist.txt" \
  "$HOME/.openclaw/workspace/main/bin/setup-pihole-candidate-test.sh" --enable
```

Assign only selected test devices to `Candidate-Test`. Do not assign every client. If every client receives the canary group, the test is global and no longer isolated.

Do not modify unrelated production groups such as Balanced, Strict, Device, Policy, Work, or a default block group.

## 11. Test reviewed candidates

Add a reviewed `block` decision to `recommendation-feedback.csv` with an empty `test_completed_at`.

Always preview first:

```sh
"$HOME/.openclaw/workspace/main/bin/sync-candidate-test-rules.sh"
```

The dry run should print the exact proposed domains and make no API change.

Apply only after reviewing the preview:

```sh
"$HOME/.openclaw/workspace/main/bin/sync-candidate-test-rules.sh" --apply
```

Test normal device functions for at least 72 hours. Seven days is better for services used less often.

Check:

- Sign-in and account functions.
- Notifications.
- Streaming and media.
- Smart-home control.
- Firmware and software updates.
- Location services.
- Payments and purchases.
- Remote access.
- Mobile and desktop behavior.

If a domain causes breakage, remove the test rule or group assignment and document the result.

When testing succeeds, set `test_completed_at` and export the approved set:

```sh
"$HOME/.openclaw/workspace/main/bin/export-approved-denylist.sh"
```

Publish only the exported domain names. Never publish the feedback notes, query counts, client data, report, or test-device identity.

## 12. Use the correct Pi-hole object type

Do not confuse local rules with subscribed lists.

Use an exact local rule when:

- The request is for one explicit domain.
- The rule needs a narrow group assignment.
- The rule is an exception that should remain local.

Use a subscribed list when:

- The user supplied or approved a domain file.
- The set needs version control.
- The set should be independently removable.
- Pi-hole should refresh it through gravity.

A local file cannot be a Pi-hole subscription. Publish the validated file at a stable HTTPS URL first.

For a Pi-hole v6 blocklist subscription:

1. Validate that entries are valid, unique, normalized, and sorted.
2. Publish the list.
3. Verify the hosted file matches the reviewed local file.
4. `POST /api/lists?type=block` with `address`, `comment`, `groups`, and `enabled`.
5. Run `POST /api/action/gravity`.
6. Verify subscription counts and imported-domain counts.
7. Confirm representative domains are blocked through DNS.

Preserve existing subscriptions. A new candidate feed should be an independent rollback boundary.

## 13. Promote allow rules safely

Permanent functional exceptions should normally live in a maintained allowlist instead of accumulating as brittle exact rules.

Use this sequence:

1. Audit the current exact allow rules and group assignments.
2. Classify permanent functional exceptions.
3. Preserve scoped exceptions that should remain local or group-specific.
4. Add the permanent entries to the maintained allowlist source.
5. Validate and publish the generated allowlist.
6. Run Pi-hole gravity.
7. Verify each domain has maintained-list coverage through the API.
8. Delete only the exact rules that were successfully replaced.
9. Verify DNS behavior and scoped exceptions again.

Never delete exact allow rules before the published replacement is imported and verified.

## 14. Verify every live change

A successful API response is not enough.

For exact allow or deny rules, verify:

- Domain.
- Rule kind and type.
- Enabled state.
- Group assignment.
- Comment or change tag.
- DNS behavior.

For subscribed lists, verify:

- Stable source URL.
- Enabled state.
- Group assignment.
- Before and after list counts.
- Gravity completion.
- Imported-domain count.
- Representative DNS behavior.

Tag temporary or batch-created rules with a unique comment. If rollback is required, remove only rules carrying that tag.

## 15. Configure optional email delivery

Email is optional and disabled by default.

The sender should accept configuration through environment variables:

| Variable | Purpose |
|---|---|
| `PIHOLE_SMTP_USER` | SMTP account |
| `PIHOLE_SMTP_PASSWORD_FILE` | Local app-password file |
| `PIHOLE_REPORT_RECIPIENT` | Recipient |
| `PIHOLE_DOMAIN_REPORT_FILE` | Report to attach |
| `PIHOLE_SMTP_URL` | Provider SMTP endpoint |

Do not ship personal addresses or a provider-specific account as defaults.

Store the SMTP app password locally:

```sh
umask 077
printf 'Paste the SMTP app password: ' >&2
IFS= read -r -s SMTP_SECRET
printf '\n' >&2
printf '%s' "$SMTP_SECRET" > "$HOME/.openclaw/service-env/smtp-app-password"
unset SMTP_SECRET
chmod 600 "$HOME/.openclaw/service-env/smtp-app-password"
```

Run the report generator manually and inspect its output before enabling email. Do not send a real email merely to validate the report generator or scheduler.

## 16. Schedule monitoring on macOS

Use one LaunchAgent for the frequent monitor and one for the daily review. Reuse existing jobs when present.

List possible existing jobs first:

```sh
find "$HOME/Library/LaunchAgents" -type f \
  \( -iname '*openclaw*.plist' -o -iname '*pihole*.plist' \) \
  -print
```

Inspect a candidate:

```sh
plutil -p "$HOME/Library/LaunchAgents/EXISTING_FILE.plist"
```

Before changing a plist:

1. Copy it to a timestamped backup in the same local directory.
2. Preserve its label, paths, schedule, logs, and unrelated environment variables.
3. Modify it atomically.
4. Run `plutil -lint`.
5. Reload only that user LaunchAgent.
6. Verify it with `launchctl print`.

Example five-minute monitor plist:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.openclaw.pihole-monitor</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>USER_HOME/.openclaw/workspace/main/bin/monitor-pihole.sh</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PIHOLE_HOST</key>
    <string>PIHOLE_IP_OR_HOSTNAME</string>
  </dict>
  <key>StartInterval</key>
  <integer>300</integer>
  <key>StandardOutPath</key>
  <string>USER_HOME/.openclaw/workspace/main/memory/pihole-monitor.stdout.log</string>
  <key>StandardErrorPath</key>
  <string>USER_HOME/.openclaw/workspace/main/memory/pihole-monitor.stderr.log</string>
</dict>
</plist>
```

Replace `USER_HOME` with the actual absolute home path. LaunchAgents do not expand `$HOME` inside string values.

Example daily schedule keys:

```xml
<key>StartCalendarInterval</key>
<dict>
  <key>Hour</key>
  <integer>6</integer>
  <key>Minute</key>
  <integer>0</integer>
</dict>
```

If the UniFi console uses a trusted local self-signed certificate and the exception is required, put `UNIFI_SKIP_CERTIFICATE_CHECK=true` in the daily job's `EnvironmentVariables` dictionary. Do not apply that setting globally.

Load and verify a new or updated user job:

```sh
PLIST="$HOME/Library/LaunchAgents/local.openclaw.pihole-monitor.plist"
plutil -lint "$PLIST"
launchctl bootout "gui/$(id -u)" "$PLIST" 2>/dev/null || true
launchctl bootstrap "gui/$(id -u)" "$PLIST"
launchctl print "gui/$(id -u)/local.openclaw.pihole-monitor"
```

If an existing job cannot be identified confidently, stop. Do not invent and load another job.

## 17. Schedule on Linux

Use a user-level systemd service and timer or the site's approved scheduler. Keep secrets in files, not directly in unit files.

The service should call the wrapper with an explicit Pi-hole host:

```ini
[Service]
Type=oneshot
Environment=PIHOLE_HOST=PIHOLE_IP_OR_HOSTNAME
ExecStart=%h/.openclaw/workspace/main/bin/run-daily-review.sh
```

Use a timer for the desired local schedule. Validate with:

```sh
systemctl --user daemon-reload
systemctl --user enable --now YOUR_TIMER.timer
systemctl --user status YOUR_TIMER.timer
journalctl --user -u YOUR_SERVICE.service --no-pager
```

## 18. Troubleshoot Pi-hole FTL correctly

Pi-hole DNS can remain healthy while the embedded web and API service is hung. A successful DNS query does not prove the API is healthy.

Check on the Pi-hole host:

```sh
sudo systemctl status pihole-FTL --no-pager
sudo ss -lntup
curl -k -I --max-time 5 https://127.0.0.1/admin/
sudo journalctl -u pihole-FTL --since '1 hour ago' --no-pager
```

After a normal restart, confirm the PID and start timestamp changed. If systemd reports success but the old process remains, schedule disruptive work for an approved maintenance window. A forced process termination can briefly interrupt DNS.

Do not restart or kill FTL during working hours without explicit approval.

Direct Pi-hole API calls are often more reliable than browser automation for this workflow, especially with self-signed certificates.

## 19. Troubleshoot common setup failures

### Authentication fails

- Confirm the app-password file exists and is not empty.
- Confirm the file contains one line and no copied label.
- Do not print the password.
- Confirm the Pi-hole host is correct.
- Confirm `POST /api/auth` returns a session SID.

### DNS works but the API times out

- Treat DNS and the FTL web/API service as separate health checks.
- Inspect `systemctl`, listening sockets, localhost HTTPS, and the FTL journal.
- Confirm a restart actually changed the process.

### UniFi collection returns pending

- Confirm both local files exist.
- Confirm the API-key mode is `400` or `600`.
- Confirm the base URL is an HTTPS origin without an API path.
- Confirm the key is valid for the local Network application.
- Do not weaken redirect or write-method protections.

### Client counts differ between runs

- The UniFi endpoint returns currently connected clients.
- Wireless devices can disconnect or rotate MAC addresses.
- Treat normal count changes as expected unless collection itself fails.

### Correlation is incomplete

- Normalize MAC addresses before comparison.
- Match MAC first and current IP second.
- Check for randomized Wi-Fi MAC addresses.
- Flag conflicts instead of guessing.

### A candidate list affects every device

- Check whether every client belongs to `Candidate-Test`.
- Remove unintended group assignments.
- Confirm the candidate list is not also assigned to the production default group.

### A blocklist file became many regex rules

- Stop and identify the intended Pi-hole object type.
- Remove only rules tagged by the mistaken rollout.
- Publish the file at a stable URL.
- Add it as a separate subscription.
- Run gravity and verify.

### Shell validation fails on Windows

Use the Git for Windows shell directly:

```powershell
& 'C:\Program Files\Git\usr\bin\sh.exe' -n .\script-name.sh
```

## 20. Rollback

### Monitor and report rollback

- Unload only the related scheduled job.
- Restore the timestamped scheduler backup.
- Keep private reports for diagnosis or remove them locally under the site's retention policy.

### UniFi rollback

- Unload the collector schedule.
- Remove the local inventory.
- Revoke the API key in UniFi.
- Remove the local key and base-URL files.

### Candidate-test rollback

- Remove test clients from `Candidate-Test`.
- Disable the candidate subscription.
- Run gravity.
- Remove only temporary exact rules created by the candidate workflow.
- Leave production groups and lists unchanged.

### Subscribed-list rollback

- Disable or remove only the new subscription.
- Run gravity.
- Verify representative domains again.

### Exact-rule rollback

- Remove only rules with the change's unique comment or tag.
- Re-read the API state.
- Verify DNS behavior.

## 21. Validation checklist

The base installation is complete only when:

- [ ] No personal value or credential is committed.
- [ ] All shell scripts pass `/bin/sh -n`.
- [ ] Runtime directories are mode `700`.
- [ ] Credential files are mode `600` or `400`.
- [ ] Pi-hole web and DNS checks both pass.
- [ ] Pi-hole v6 authentication succeeds without exposing the password.
- [ ] The API session is closed after each run.
- [ ] Metrics stay local.
- [ ] Curated lists download from stable HTTPS URLs.
- [ ] Protected-domain rules are loaded.
- [ ] Domain descriptions have no duplicate keys.
- [ ] Recommendation tables contain no unexplained fallback descriptions.
- [ ] The daily report states that it is recommendation-only.
- [ ] The daily report does not change Pi-hole policy.
- [ ] UniFi collection, if enabled, uses GET only.
- [ ] The UniFi collector refuses insecure key permissions.
- [ ] The UniFi collector does not follow redirects with the API key.
- [ ] The UniFi inventory remains local and mode `600`.
- [ ] Correlation uses MAC first and IP second.
- [ ] Scheduler configuration is valid and loaded once.
- [ ] Email remains disabled until explicitly configured and tested.
- [ ] Candidate testing is isolated to selected devices.
- [ ] Every live change has API readback, DNS verification, and rollback steps.

## Operating cadence

### Every five minutes

- Run the health monitor.
- Record transitions and summary metrics.
- Alert only on a new failure or recovery.

### Daily

- Refresh public comparison lists.
- Refresh the reduced UniFi inventory.
- Generate the 24-hour Pi-hole review.
- Review coverage issues, block candidates, and unblock candidates.
- Do not automatically modify policy.

### Weekly

- Review unresolved recommendations.
- Check canary devices for breakage.
- Review scheduler errors and stale inventory.
- Check credential-file permissions.

### After a completed canary test

- Record the completion timestamp.
- Export the approved domains.
- Review the public diff.
- Publish through the normal pull-request process.
- Verify the hosted artifact before subscribing Pi-hole to it.

## Public repository rules

Safe to publish:

- Scripts with generic defaults.
- Configuration templates.
- Public domain lists.
- Sanitized fixtures.
- Documentation.
- Count-only test results.

Never publish:

- Pi-hole passwords or session IDs.
- UniFi API keys.
- SMTP credentials.
- Raw DNS queries.
- Client names.
- MAC addresses.
- Private IP addresses.
- Internal hostnames.
- Site UUIDs.
- Generated inventories.
- Local reports.
- Scheduler backups.

## Design decisions

These choices are intentional:

- Read-only collection is automated. Policy mutation is not.
- UniFi access is reduced to a local inventory file before OpenClaw uses it.
- Recommendation confidence describes evidence, not safety.
- Domain descriptions remain cautious and explain possible impact.
- Broad platform domains are protected from automatic suggestions.
- Canary tests use a separate group and separate list.
- Subscribed lists and local exact rules are treated as different objects.
- Published lists contain domains only.
- Exact allows are migrated only after maintained-list coverage is verified.
- Operational interruptions are scheduled around user availability.
- Success requires state verification, not only a successful command.

## License

This project is licensed under the [MIT License](LICENSE).

## Contributing and support

- Read [CONTRIBUTING.md](CONTRIBUTING.md) before proposing a change.
- Use [SUPPORT.md](SUPPORT.md) for setup questions and troubleshooting guidance.
- Report suspected security vulnerabilities according to [SECURITY.md](SECURITY.md).
