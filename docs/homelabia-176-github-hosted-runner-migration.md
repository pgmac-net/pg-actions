# homelabia#176: Move CI validation off self-hosted runners

Ticket: [pgmac-net/homelabia#176](https://github.com/pgmac-net/homelabia/issues/176)

## Overview

Self-hosted ARC runners on `pvek8s` were executing every CI job across the estate, including work with no homelab dependency at all — lint, `terraform validate`/`plan` against SaaS backends, static and security scans. This moved everything portable onto GitHub-hosted runners, leaving self-hosted for jobs that genuinely need homelab-only infrastructure.

**Outcome: 76.3% of measured self-hosted execution (970 of 1,272 min) now runs on GitHub-hosted runners. Zero VALIDATION jobs remain on self-hosted.**

## What counts as "must stay self-hosted"

A job stays only if it reaches something unreachable from outside the homelab:

- internal registry `macro.int.pgmac.net:5000`
- internal Dependency-Track `dtrack.int.pgmac.net`
- BuildKit over TCP to `buildkitd.arc-runners.svc.cluster.local`
- an ansible inventory reachable over the homelab network
- the live `pvek8s` cluster, or Proxmox at `pve2.int.pgmac.net`

## Runner tiering

`ubuntu-slim` bills at $0.002/min against `ubuntu-latest`'s $0.006/min, but runs **unprivileged with no Docker** and minimal preinstalled software.

| Runner | Used for |
|---|---|
| `ubuntu-slim` | terraform fmt/validate/plan/apply, mkdocs, zizmor, Trivy config/fs scans, Slack notify, ruff/pylint |
| `ubuntu-latest` | Jobs needing `jq`/`curl`/`npm`/`gh`/`grype`, or Graphviz |
| `self-hosted` | Anything touching homelab-only infrastructure |

## Key findings

### Trivy is a composite action, not a Docker action

`aquasecurity/trivy-action` uses `runs: using: composite` — it installs the Trivy binary rather than running as a container. Every Trivy job in the estate uses `scan-type: config` or `fs`, so all of them run on `ubuntu-slim`. The exception is `budgeteer`'s SBOM job, which uses `image-ref` against the internal registry — an image scan needing both Docker and homelab access, so it correctly stays self-hosted.

### `ubuntu-slim` cannot host a job container

Both `slack-notify` reusables declared `container: ubuntu:26.04`, which would have hard-failed on slim. Per PGM-169, that directive was added in pg-actions #55 only because ARC runner 2.334 set `ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER=true` — a self-hosted-only constraint, already neutralised by setting that env var to `false` on the runnersets, and irrelevant on GitHub-hosted runners. The containers were dropped rather than downgrading to `ubuntu-latest`.

### Renovate looked portable but is not

`pgk8s/renovatebot.yml` is just `npx renovate` with a GitHub token — nothing homelab-specific in the workflow. The dependency lives in `renovate.json`: its `customManagers` extract `registryUrl=`, and the repo carries `registryUrl=macro.int.pgmac.net:5000` annotations plus a helm datasource at `macro.int.pgmac.net/charts/`. Renovate must reach both to resolve versions. **Classification cannot be done from the workflow body alone.**

### A job can be half-portable

`slack-weather`'s `check` job linted *and* build-checked. The lint half is portable; the build half used BuildKit-over-TCP and a `cache-from` pointing at the internal registry. It was split into `lint` (slim) and `build-check` (self-hosted) rather than moved wholesale.

### SHA-pinned callers do not inherit reusable-workflow changes

Changing `runs-on` in a `pg-actions` reusable propagates automatically only to callers using `@main`. Five repos pinned `sbom.yml` to a commit SHA and kept running the old self-hosted version — caught only by checking a post-merge run's job labels. Their pins were bumped to `935d4b8` in a follow-up wave, keeping the pin (deliberate movement, not floating).

### Migration surfaces latent failures

`slack-weather`'s lint went red on three pre-existing ruff findings, reproducible locally. The job simply had not run recently enough to surface them. **A red PR during a runner migration does not automatically mean the migration broke something.**

## Delivery

Four waves, 24 PRs.

| Wave | Scope | PRs |
|---|---|---|
| 1 — canary | `pg-actions` reusables (fan-out) + `incidents` | 2 |
| 2 — fan-out | 10 repos | 10 |
| 2b — pin bumps | SHA-pinned `sbom.yml` callers | 5 |
| 3 — production applies | `terraform-cloudflare-config`, `terraform-github` | 2 |
| 4 — annotations | Comment-only, why each retained job stays | 10 |

The canary wave was deliberately public repos with high run volume, so a systemic mistake would surface on free minutes across 2 PRs rather than 17. It caught the job-container problem immediately.

## Measured result

Baseline: 791 self-hosted jobs / 1,272 min execution, sampled at job level across the estate.

| Fate | Jobs | Execution | Share |
|---|---|---|---|
| Moved to `ubuntu-slim` | 517 | 804.3 min | 63.2% |
| Moved to `ubuntu-latest` | 39 | 165.9 min | 13.0% |
| Stays self-hosted (DEPLOY) | 210 | 295.0 min | 23.2% |

Secondary win: `terraform-module-github-repo`'s 6-way matrix averaged **4.3–4.8 min queued** on self-hosted, peaking over 25 min, against ~1.3 min of actual execution per leg — each leg wanted its own ARC pod. On GitHub-hosted runners that contention disappears.

Cost impact is negligible: before this work only `pgk8s` consumed billed private-repo minutes (~527/mo) at $0.00 net, well inside the included quota. Public repos are free regardless.

## Corrections made during the work

Two claims in the original issue body were wrong and were corrected on the ticket:

1. **Dependabot "Graph Update" runs stuck at the 24h timeout were attributed to self-hosted runner starvation.** They carry the `dependabot` label with an empty `runner_name` — GitHub's own Dependabot infrastructure, never touching ARC. Split out as [homelabia#177](https://github.com/pgmac-net/homelabia/issues/177).
2. **Self-hosted runners were described as slow/queue-bound.** Job-level data showed median queue in *seconds*. The real contention is narrow: matrix fan-outs. The case for this work is load reduction on pvek8s, not latency.

## Verification gap

The `slack-notify-start`/`slack-notify-end` reusables are only invoked via `workflow_call`, and no calling workflow fired between merge and write-up — so their move to `ubuntu-slim` is **unverified in production**. Specific risk: both use `date --iso-8601=minutes`, which PGM-169 flagged when ruling out Alpine (busybox `date` lacks it). `ubuntu-slim` is an Ubuntu userland with GNU coreutils, so this should be fine, but it has not been observed. Verify by dispatching any workflow that calls them (`ansible/dns.yml`, `docker-ghar`, `docker-gnuplot`, `Docker-Nagios/docker-image.yml`).

Also outstanding: `ansible/diagrams.yml` should be dispatched once. It may have been broken before this work — it installs `diagrams` (needing Graphviz `dot`) with no apt install, inside a container with no Graphviz, while its own SVG step rewrites a `/opt/hostedtoolcache` path that only exists on GitHub-hosted runners.

## Follow-ups

- [homelabia#177](https://github.com/pgmac-net/homelabia/issues/177) — Dependabot dependency-graph runs stuck queued on GitHub infrastructure
- [homelabia#178](https://github.com/pgmac-net/homelabia/issues/178) — workflows still creating Linear tickets after decommissioning
