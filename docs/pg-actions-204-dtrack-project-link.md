# pg-actions#204: Dependency Track link in the SBOM scan summary

Ticket: pgmac-net/pg-actions#204 · PR: pgmac-net/pg-actions#205

## Problem

`sbom.yml` passed the Dependency Track project URL between jobs as the output `dtrack-project-url`. The URL contains the `DT_HOST` secret, so GitHub dropped the output (`Skip output 'dtrack-project-url' since it may contain secret`). The scan summary and PR comment never got the link.

## Change

- `DepTrack-sbom` outputs only the non-secret project UUID as `dtrack-project-uuid`.
- The UUID comes from the upload response (`projectUuid` in `PUT /api/v1/bom`). The project lookup planned in the ticket was dropped: it ran right after the upload and missed while Dependency Track was still processing the BOM.
- `GHAS-dependency-scan` rebuilds `https://${DT_HOST}/projects/<uuid>` from `secrets.DT_HOST`. It adds the link only when both the host and the UUID are set, so Dependabot and fork PRs skip it.
- If the upload response has no `projectUuid`, the step emits a warning and skips the link. The upload itself still fails hard.

## Caller impact

None. `dtrack-project-uuid` is a job output, not a `workflow_call` output, so callers don't see it. Callers get the link after bumping their pin to a commit that includes this change.

## Verification

- actionlint clean; zizmor no findings.
- PR #205 run `45781107921` (head `34f0c0a`): `dtrack-project-uuid` set, no `Skip output` warning, and the PR comment links to `https://dtrack.int.pgmac.net/projects/db245cf4-…`.
