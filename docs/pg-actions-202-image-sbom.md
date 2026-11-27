# pg-actions#202: image SBoMs from CI (`image-sbom.yml`)

Ticket: [pgmac-net/pg-actions#202](https://github.com/pgmac-net/pg-actions/issues/202). Plan:
[issuecomment-7228815389](https://github.com/pgmac-net/pg-actions/issues/202#issuecomment-7228815389).
Part of the CVE checker epic [homelabia#267](https://github.com/pgmac-net/homelabia/issues/267):
the naming contract comes from [homelabia#292](https://github.com/pgmac-net/homelabia/issues/292).

Three repositories built container images and SBoM'd them with three different steps (budgeteer:
trivy + `gh-upload-sbom`; Docker-Nagios: syft + a stored `DT_PROJECT_UUID`; metasearch: syft +
`gh-upload-sbom`, into the repo's own project). The daily cluster scanner (`sbom-scanner images`)
already SBOMs every running digest, so CI's job is a build-time SBoM under the same name.

## Decisions (grilling, 2026-11-28)

1. **A separate reusable workflow, `image-sbom.yml`**, not an input on `sbom.yml`: that one is built
   around a directory (syft `./`, dependency snapshot, grype SARIF, repo naming).
2. **trivy**, pinned to the cluster scanner's version (0.76.2), so a digest uploaded by CI and
   later re-scanned by the scanner gives the same BOM.
3. **Attest the image digest with its SBoM** (`actions/attest-sbom`), **public callers only**: step
   `if: github.event.repository.visibility == 'public'`, the same gate `sbom.yml` uses. In a
   reusable workflow the `github` context is the caller's, so private budgeteer skips it with no
   configuration.
4. **No deletion of the old DT projects** here; the callers just stop uploading to them.
5. **No gating in `image-sbom.yml`.** budgeteer keeps its CRITICAL/HIGH trivy gate as a separate
   job beside the SBoM job, so a failing gate still fails the run but never stops the pushed image
   getting its SBoM.

## Naming contract

| | |
|---|---|
| Project name | the fully qualified image repository, e.g. `macro.int.pgmac.net:5000/budgeteer` |
| Project version | `<tag>@sha256:<digest>` |
| Tags | `source:image`, `image:<repository>`; exposure is set by the cluster scanner from the Cloudflare tunnel |

A digest CI uploads gets the same project and version the cluster scanner would give it, so the
scanner finds a fresh version and skips it.

## What was built

| Where | What |
|---|---|
| `pg-actions` [#214](https://github.com/pgmac-net/pg-actions/pull/214) | `image-sbom.yml` and its README section |
| `budgeteer` [#132](https://github.com/pgmac-net/budgeteer/pull/132) | `SBOM` job calls the workflow; `Build` outputs the digest; the trivy gate is its own `Scan` job |
| `Docker-Nagios` [#23](https://github.com/pgmac-net/Docker-Nagios/pull/23) | syft, attest and `DT_PROJECT_UUID` upload replaced by a call; `build-n-push` outputs digest and tag |
| `metasearch` [#41](https://github.com/pgmac-net/metasearch/pull/41) | `sbom` job replaced by a call, on tag builds only |

The workflow validates its inputs (repository, `sha256:` digest, tag) before they reach a shell or
the API, drops exact duplicate components (Dependency-Track 5 rejects them), uploads with
`isLatest` and `isActive`, and never uploads for `pull_request` events.

## Deviations from the plan

- **No `gh attestation verify` step.** The registry is internal and the attestation is not pushed
  to it, so there is no image reference to verify against. The attestation is still created and
  stored on GitHub.
- **trivy runs through `aquasecurity/setup-trivy` and a direct `trivy image`**, not `trivy-action`:
  v0.36.0 of the action has no inputs for the flags needed (skip DB update, insecure registry).
- **metasearch uploads on tag builds only.** The old job also ran on every push to master, putting
  a version per merge into DT; master images are never deployed. One-line change if master builds
  should be recorded too.
- **Secrets are mapped explicitly** (`DT_HOST`, `DT_APIKEY`), not `secrets: inherit`, after zizmor
  flagged `secrets-inherit` on the first budgeteer draft.
- A UTF-8 byte-order mark that my editor wrote into the first draft (YAML-invalid; zizmor refused
  to load the file) was found by bisecting and removed before the PR.

## Verification

Checked: actionlint and zizmor clean on the new workflow; zizmor shows no new findings on the three
callers; the trivy, de-duplicate and payload steps run locally on a real digest (`alpine`:
CycloneDX 1.7, 17 components, payload with the contract name, version and tags).

**Not yet run: the workflow itself.** The builds run on `x.y.z` tags (budgeteer, Docker-Nagios) or
`v*` tags (metasearch). Proof for each is the next release: a project
`<registry>/<image>` / `<tag>@sha256:…` with a BOM appears in Dependency-Track, and the next
cluster scan logs it as not needing a scan.

## Follow-ups (owner, after the proof runs)

- Delete the old repo-named image projects in Dependency-Track: `budgeteer` 2.8.1, `Docker-Nagios`,
  and the `0.3.x` tag versions inside `pgmac-net/metasearch` (one version at a time).
- Remove the unused secrets: `DT_PROJECT_UUID` on Docker-Nagios, `DEPENDENCYTRACK_HOSTNAME` and
  `DEPENDENCYTRACK_APIKEY` on budgeteer.
- Docker-Nagios still reads `inputs.tags` without declaring it (actionlint flags it) and uses
  `actions/checkout@v3`; neither is changed here.
