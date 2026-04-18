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
