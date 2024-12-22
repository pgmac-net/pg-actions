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
- Save the results (as a SARIF file) to the Security tab of your repo

A second job can:

- Pull down the SBoM artefact
- Verify the attestation
- Scan the SBoM for any known vulnerabilities in your dependencies
- Output the result (as a text table) to the workflow output logs
