# Linear → GitHub Issues (homelabia#178)

Linear is being decommissioned in favour of GitHub Issues. Three workflows across
three repos were still creating Linear tickets, so drift detections and Dependabot
findings were being filed somewhere that is being switched off.

| Repo | Workflow | Job |
|---|---|---|
| `pgmac-net/pg-actions` | `dependabot-management.yml` | `create-linear-ticket` → `create-issue` |
| `pgmac-net/terraform-github` | `drift-detect.yml` | `drift-detect` |
| `pgmac-net/terraform-cloudflare-config` | `drift-detect.yml` | `drift-detect` |

## Where issues are filed

Each workflow files into `$GITHUB_REPOSITORY` — the repo it is running in — using
`github.token` plus a job-level `issues: write` grant. Dependabot findings land in
the repo that owns the vulnerable dependency, which is also where the fix PR lands;
drift lands in the terraform repo that drifted.

The alternative, filing everything into `pgmac-net/homelabia` for a single queue,
was rejected: `github.token` cannot write across repos, so it would have needed
`GH_BOT_TOKEN` in every managed repo, and `dependabot-management.yml` is a reusable
workflow callable from repos that may not carry that secret.

## Labels

The three code repos only carry GitHub's default label set, so every label the new
issues use is created on demand — `gh label create ... || true` in shell, or
`createLabel` in a try/catch that swallows HTTP 422 in `github-script`. This is
idempotent and self-healing in any repo.

| Label | Colour | Used by |
|---|---|---|
| `security` | `d73a4a` | Dependabot issues |
| `dependencies` | `0366d6` | Dependabot issues |
| `severity:critical` | `b60205` | Dependabot issues (replaces Linear priority 1) |
| `severity:high` | `d93f0b` | Dependabot issues (replaces Linear priority 2) |
| `drift` | `fbca04` | Drift issues |

`terraform-github` manages repo configuration as IaC but does **not** manage labels,
so creating these does not cause config drift.

## Behaviour changes

These are deliberate improvements, not straight translations of the Linear logic.

**Dependabot dedup is open-only.** The Linear query matched tickets in any state, so
a closed ticket permanently silenced that GHSA for that repo. The replacement searches
`--state open`, meaning a still-open alert whose issue was closed gets a fresh issue —
the vulnerability is still there, so that is the correct signal.

**Drift dedup now exists.** Previously each weekly run created a new dated ticket, so
drift left unresolved for six weeks produced six tickets. The step now looks for an
open issue labelled `drift`, comments on it with the new plan output and run URL, and
only creates an issue when none is open. Closing the issue lets the next run open a
fresh one. `listForRepo` also returns pull requests, so the lookup filters those out.

**The fix PR closes its issue.** The PR body carries `Closes #N` instead of a Linear
backlink. A merged draft fix PR therefore closes the tracking issue even if the fix
was incomplete; the Dependabot alert itself remains the source of truth.

## Why `github-script` in the drift workflows

Both drift jobs run on `ubuntu-slim`, which the homelabia#176 migration write-up
describes as having minimal preinstalled software. Neither `gh` nor `jq` is guaranteed
there, and the Linear step's existing `curl`+`jq` usage had never actually fired in
production — drift had not been detected since the move. `actions/github-script` is a
pure JS action with no dependency on the runner's shell toolset, which removes the
assumption entirely.

`dependabot-management.yml` runs on `ubuntu-latest`, where `gh` is guaranteed, so it
uses the `gh` CLI and stays shell-shaped like the rest of that file.

Building the drift body in JavaScript also fixed a latent bug: the old shell string was
written as an indented multi-line literal, so every line of the ticket description
carried leading whitespace.

## Propagation dependency

`terraform-github/repository_files.tf` reads `.github/workflows/dependabot-alert.yml`
from `pg-actions@main` and writes it into every managed repo. The orchestrator gained
`issues: write`, so editing the source is not sufficient — until a `terraform-github`
apply runs, stale copies in managed repos lack the permission and the reusable
workflow's `gh issue create` returns 403.

Ordering: merge pg-actions → run the terraform-github apply → merge the two drift PRs.

## `LINEAR_API_KEY`

Removed from `dependabot-management.yml`'s `workflow_call.secrets` block, both drift
workflows, `terraform-github/scripts/set-claude-secrets.sh`, and both README secrets
tables. The stored org and per-repo secrets are deleted separately, after the
verification dispatches confirm nothing else consumes the key.

## Verification

None of these paths run on a pull request — drift is schedule/dispatch-only and
`dependabot-management.yml` is `workflow_call`-only — so a green PR check proves
nothing about the issue-creation code. `actionlint` and `zizmor` were run against every
changed file locally, with findings compared against the pre-change baseline to confirm
no regressions. Real verification is a post-merge `workflow_dispatch` of all three
entry points.
