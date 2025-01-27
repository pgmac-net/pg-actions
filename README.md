# pg-actions

Where I store my re-usable actions

In here, there is:

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
