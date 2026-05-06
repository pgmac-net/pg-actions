# pg-actions

Where I store my re-usable actions

In here, there is:

# Slack Notifications

Two reusable workflows that handle Slack notifications for GitHub Actions runs. They post a start message when a workflow begins and update it with the final status, timing, and colour when it finishes.

## slack-notify-start

Posts an initial "In Progress" Slack message via `chat.postMessage` and outputs the timing values needed by `slack-notify-end`.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `channel` | No | `C01UGQ98P9U` | Slack channel ID to post to |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `SLACK_BOT_TOKEN` | Yes | Slack bot token with `chat:write` permission |

### Outputs

| Name | Description |
|------|-------------|
| `ts` | Slack message timestamp — pass to `slack-notify-end` |
| `start` | Start time as epoch seconds — pass to `slack-notify-end` |
| `startfmt` | Start time formatted as ISO-8601 — pass to `slack-notify-end` |

### Usage

```yaml
jobs:
  slack-start:
    uses: pgmac-net/pg-actions/.github/workflows/slack-notify-start.yml@main
    secrets: inherit

  run:
    needs: [slack-start]
    runs-on: self-hosted
    steps:
      - run: echo "do the work here"

  slack-end:
    if: always()
    needs: [slack-start, run]
    uses: pgmac-net/pg-actions/.github/workflows/slack-notify-end.yml@main
    with:
      ts: ${{ needs.slack-start.outputs.ts }}
      start: ${{ needs.slack-start.outputs.start }}
      startfmt: ${{ needs.slack-start.outputs.startfmt }}
      status: ${{ needs.run.result }}
    secrets: inherit
```

For multi-job workflows where you want the overall status to reflect several parallel jobs:

```yaml
  slack-end:
    if: always()
    needs: [slack-start, job-a, job-b, job-c]
    uses: pgmac-net/pg-actions/.github/workflows/slack-notify-end.yml@main
    with:
      ts: ${{ needs.slack-start.outputs.ts }}
      start: ${{ needs.slack-start.outputs.start }}
      startfmt: ${{ needs.slack-start.outputs.startfmt }}
      status: ${{ (needs.job-a.result == 'failure' || needs.job-b.result == 'failure' || needs.job-c.result == 'failure') && 'failure' || (needs.job-a.result == 'cancelled' || needs.job-b.result == 'cancelled' || needs.job-c.result == 'cancelled') && 'cancelled' || 'success' }}
    secrets: inherit
```

## slack-notify-end

Updates the Slack message posted by `slack-notify-start` with the final status, colour, and timing information.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `channel` | No | `C01UGQ98P9U` | Slack channel ID |
| `ts` | Yes | — | Slack message timestamp from `slack-notify-start` |
| `start` | Yes | — | Start time as epoch seconds from `slack-notify-start` |
| `startfmt` | Yes | — | Start time formatted as ISO-8601 from `slack-notify-start` |
| `status` | Yes | — | Overall job status: `success`, `failure`, or `cancelled` |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `SLACK_BOT_TOKEN` | Yes | Slack bot token with `chat:write` permission |

### Status colours

| Status | Slack colour | Icon |
|--------|-------------|------|
| `success` | `good` (green) | ✅ |
| `failure` | `danger` (red) | 🛑 |
| `cancelled` | `warning` (yellow) | ⚠ |
| anything else | `grey` | ❓ |

---

# OpenSSF Scorecard

Two reusable workflows that run [OpenSSF Scorecard](https://scorecard.dev/) security checks against your repository. Results are kept local — nothing is published to the OpenSSF public API.

## scorecard-public

For **public** repositories. Runs the full Scorecard check suite, uploads results as an artifact, and uploads the SARIF report to the GitHub Security tab.

Also runs automatically on push to `main`/`master` and on a weekly schedule (Sunday midnight AEST) when used directly in `pg-actions`.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `runs-on` | No | `self-hosted` | Runner label |

### Permissions required in caller

```yaml
permissions:
  security-events: write
  contents: read
  actions: read
```

### Usage

```yaml
jobs:
  scorecard:
    uses: pgmac-net/pg-actions/.github/workflows/scorecard-public.yml@main
```

---

## scorecard-private

For **private** repositories. Runs Scorecard checks and uploads results as an artifact only. The GitHub Security tab upload is skipped (GHAS code scanning requires a paid plan for private repos).

Only callable via `workflow_call` — no standalone schedule trigger.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `runs-on` | No | `self-hosted` | Runner label |

### Permissions required in caller

```yaml
permissions:
  contents: read
  actions: read
```

### Usage

```yaml
jobs:
  scorecard:
    uses: pgmac-net/pg-actions/.github/workflows/scorecard-private.yml@main
```

---

# sbom

This will:

- create a SBoM of your repo
- Attest this SBoM
- Upload the SBoM as an artefact to your repo
- Upload the SBoM to Dependency Track
- Scan the SBoM for any known vulnerabilities in your dependencies

A second job will then:

- Pull down the SBoM artefact
- Verify the artefact attestation
- Scan the SBoM for any known vulnerabilities in your dependencies
- Save the results (as a SARIF file) and upload to the GHAS Security tab of your repo, making the dependencies known vulnerabilities visible

## What this isn't

This is not a SAST of your code, or the dependencies code.
It just scans for your dependenices (via the SBoM discovery) and maps your versions against known CVE's.

This is not a version upgrading tool
Other tools are better for that.
EG:
* DependaBot
* RenovateBot (You may notice some renovatebot stuff in here)

---

# Dependabot Alert Management

Two workflows that together automate the response to Dependabot security alerts. The orchestrator (`dependabot-alert.yml`) is distributed to each managed repo and runs on a schedule; it fans out to the reusable worker (`dependabot-management.yml`) once per high/critical open alert.

For each alert the system will:

1. Check Linear for an existing ticket matching the GHSA ID and repo — skip if found
2. Create a Linear ticket in the HomeLabia project with severity-based priority
3. Check for an existing fix PR — skip if found
4. Run Claude Code on a new branch to update the vulnerable package and lock files
5. Open a draft PR linking back to the Linear ticket

## dependabot-alert.yml

The per-repo orchestrator. Runs every 6 hours (and on manual dispatch), fetches all open high/critical Dependabot alerts, and calls `dependabot-management.yml` once per alert via a matrix strategy.

This workflow is **not** callable — copy it directly into each repo's `.github/workflows/` directory. It requires no configuration; all secrets flow through via `secrets: inherit`.

### Permissions required in caller repo

```yaml
permissions:
  contents: write
  pull-requests: write
```

### Secrets required

| Name | Required | Description |
|------|----------|-------------|
| `LINEAR_API_KEY` | Yes | Linear API key for ticket creation |
| `CLAUDE_CODE_OAUTH_TOKEN` | No | Claude Code OAuth token — PR creation is skipped if absent |

### Usage

```yaml
# .github/workflows/dependabot-alert.yml
name: Dependabot Alert Management

on:
  schedule:
    - cron: '0 */6 * * *'
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  get-alerts:
    name: List High/Critical Alerts
    runs-on: self-hosted
    outputs:
      matrix: ${{ steps.alerts.outputs.matrix }}
      has_alerts: ${{ steps.alerts.outputs.has_alerts }}
    steps:
      - name: Fetch open high/critical Dependabot alerts
        id: alerts
        env:
          GH_TOKEN: ${{ github.token }}
          REPO: ${{ github.repository }}
        run: |
          ALERTS=$(gh api "/repos/${REPO}/dependabot/alerts" \
            --jq '[.[] | select(.state == "open") | select(.security_vulnerability.severity | test("^(high|critical)$"))]')
          COUNT=$(echo "$ALERTS" | jq 'length')
          echo "Found ${COUNT} high/critical open alert(s)"
          if [ "$COUNT" -eq 0 ]; then
            echo "has_alerts=false" >> "$GITHUB_OUTPUT"
            echo "matrix={\"include\":[]}" >> "$GITHUB_OUTPUT"
          else
            echo "has_alerts=true" >> "$GITHUB_OUTPUT"
            MATRIX=$(echo "$ALERTS" | jq -c '{include: [.[] | {
              alert_number: (.number | tostring),
              package_name: .dependency.package.name,
              package_ecosystem: .dependency.package.ecosystem,
              severity: .security_vulnerability.severity,
              summary: .security_advisory.summary,
              cve_id: (.security_advisory.cve_id // ""),
              ghsa_id: .security_advisory.ghsa_id,
              vulnerable_version_range: .security_vulnerability.vulnerable_version_range,
              first_patched_version: (.security_vulnerability.first_patched_version.identifier // "")
            }]}')
            echo "matrix=${MATRIX}" >> "$GITHUB_OUTPUT"
          fi

  manage:
    name: Process Alert
    needs: get-alerts
    if: needs.get-alerts.outputs.has_alerts == 'true'
    strategy:
      matrix: ${{ fromJSON(needs.get-alerts.outputs.matrix) }}
      fail-fast: false
    uses: pgmac-net/pg-actions/.github/workflows/dependabot-management.yml@main
    with:
      alert_number: ${{ matrix.alert_number }}
      package_name: ${{ matrix.package_name }}
      package_ecosystem: ${{ matrix.package_ecosystem }}
      severity: ${{ matrix.severity }}
      summary: ${{ matrix.summary }}
      cve_id: ${{ matrix.cve_id }}
      ghsa_id: ${{ matrix.ghsa_id }}
      vulnerable_version_range: ${{ matrix.vulnerable_version_range }}
      first_patched_version: ${{ matrix.first_patched_version }}
    secrets: inherit
```

---

## dependabot-management.yml

The reusable worker. Called once per alert by `dependabot-alert.yml`. Creates the Linear ticket and (when `CLAUDE_CODE_OAUTH_TOKEN` is available) opens a draft fix PR via Claude Code.

Can also be called directly from any workflow that already has alert metadata.

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `alert_number` | Yes | — | Dependabot alert number |
| `package_name` | Yes | — | Vulnerable package name |
| `package_ecosystem` | Yes | — | Package ecosystem (`npm`, `pip`, `maven`, etc.) |
| `severity` | Yes | — | `critical` or `high` |
| `summary` | Yes | — | Short vulnerability description |
| `cve_id` | No | `""` | CVE identifier if available |
| `ghsa_id` | Yes | — | GitHub Security Advisory ID (used as the dedup key) |
| `vulnerable_version_range` | Yes | — | Affected version range |
| `first_patched_version` | No | `""` | First version containing the fix |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `LINEAR_API_KEY` | Yes | Linear API key — used to create/find tickets in the HomeLabia project |
| `CLAUDE_CODE_OAUTH_TOKEN` | No | Claude Code OAuth token — PR creation is skipped if absent; manual fix is required |

### Usage

```yaml
jobs:
  manage:
    uses: pgmac-net/pg-actions/.github/workflows/dependabot-management.yml@main
    with:
      alert_number: "42"
      package_name: "lodash"
      package_ecosystem: "npm"
      severity: "critical"
      summary: "Prototype Pollution in lodash"
      ghsa_id: "GHSA-jf85-cpcp-j695"
      cve_id: "CVE-2019-10744"
      vulnerable_version_range: "< 4.17.12"
      first_patched_version: "4.17.12"
    secrets: inherit
```

### What happens without `CLAUDE_CODE_OAUTH_TOKEN`

The Linear ticket is still created. The fix PR step is skipped and the ticket URL is printed to the job log for manual action.

---

# Claude Code (@claude)

A workflow that listens for `@claude` mentions in issues, PR comments, and PR reviews and invokes [Claude Code](https://claude.ai/claude-code) to respond.

Distribute `claude.yml` to each repo that should respond to `@claude` mentions. No configuration is needed beyond the two secrets below.

### Secrets required

| Name | Required | Description |
|------|----------|-------------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Yes | Claude Code OAuth token — generate with `claude setup-token` |

### Permissions required in caller repo

```yaml
permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write
  actions: read
```

### Trigger events

| Event | Condition |
|-------|-----------|
| `issue_comment` | Comment body contains `@claude` |
| `pull_request_review_comment` | Comment body contains `@claude` |
| `pull_request_review` | Review body contains `@claude` |
| `issues` (opened, assigned) | Issue title or body contains `@claude` |

### Usage

```yaml
# .github/workflows/claude.yml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 1

      - name: Install Claude Code
        run: |
          npm install -g @anthropic-ai/claude-code
          echo "CLAUDE_BIN=$(which claude)" >> "$GITHUB_ENV"

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          path_to_claude_code_executable: ${{ env.CLAUDE_BIN }}
          additional_permissions: |
            actions: read
```

### Getting a `CLAUDE_CODE_OAUTH_TOKEN`

```bash
claude setup-token
# copies the token to stdout — add it as a repo or org secret named CLAUDE_CODE_OAUTH_TOKEN
```

The token maps to your Claude Pro/Max subscription quota, not an API key. Org-level secrets only reach public repos on the free GitHub tier; private repos need the secret set individually.
