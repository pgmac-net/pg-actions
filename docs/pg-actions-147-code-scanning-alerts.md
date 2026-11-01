# pg-actions#147 — Code scanning alerts clean-up

Ticket: pgmac-net/pg-actions#147 ("Security tab"). 39 open code-scanning alerts (38 Scorecard, 1 Grype).

## Process

Planned on the ticket after a short grilling pass. Decisions:

- Fix all file-based alerts (pinning, token permissions, Grype).
- Fix the cheap repo-level alerts (Security-Policy, SAST); dismiss the rest with reasons.
- Add the Renovate pin preset so pins stay current; keep `dependabot.yml`.
- Switch the self-call in `dependabot-alert.yml` from `@main` to a local path.

## Changes

| Alert class | Count | Change |
|---|---|---|
| Pinned-Dependencies | 22 | Every `uses:` pinned to a commit SHA with version comment; local self-call for `dependabot-management.yml`; `helpers:pinGitHubActionDigests` in `renovate.json` |
| Grype GHSA-8q5r-mmjf-575q | 1 | `claude-code-action` pinned at v1.0.266 (fix is >= 1.0.74) |
| Token-Permissions | 11 | Top-level `permissions: contents: read` on all workflows; `dependabot-alert.yml` write scopes moved to the `manage` job; `sbom.yml` over-broad job scopes removed |
| Security-Policy | 1 | `SECURITY.md` |
| SAST | 1 | `.github/workflows/zizmor.yml` (SARIF upload) |
| Fuzzing, CII-Best-Practices, CI-Tests, Code-Review, Branch-Protection | 5 | Dismissed as won't-fix after merge (single-maintainer, workflow-only repo) |

## Deviations from plan

- `anthropics/claude-code-base-action@v1` pointed at a mirror repo that has no `v1` tag (the ref was unresolvable). Replaced with `anthropics/claude-code-action/base-action@<sha>`, the canonical source of the mirror.
- Also set `persist-credentials: false` on the two `sbom.yml` checkouts (zizmor `artipacked`).

## Known remaining zizmor findings (follow-ups)

- `dangerous-triggers` on `claude.yml` (issue_comment etc.; guarded by `@claude` condition).
- `self-repository` hint in `dependabot-alert.yml` (`./` vs `$/`): left as is; `$/` is only documented for step-level `uses:` and cannot be tested on a `workflow_dispatch` workflow. Dismiss as won't-fix.

Fixed after PR review: `secrets-inherit` in `dependabot-alert.yml` now passes `CLAUDE_CODE_OAUTH_TOKEN` explicitly.
- `artipacked` in `claude.yml` and `dependabot-management.yml` (credentials needed to push).
- `adhoc-packages` for `npm install -g` in `claude.yml`.

## Follow-up (pgmac-net/pg-actions#149)

`claude.yml` was removed (the `@claude` workflow is no longer used), which clears the `dangerous-triggers`, `artipacked` and `adhoc-packages` zizmor alerts for it plus the Scorecard unpinned-npm alert. `dependabot.yml` gained a 7-day cooldown (`dependabot-cooldown`). The `artipacked` finding on `dependabot-management.yml` is dismissed as won't-fix (the job pushes with checkout credentials). Org-wide removal of the workflow is tracked in pgmac-net/homelabia#228.
