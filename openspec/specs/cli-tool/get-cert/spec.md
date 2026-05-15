## ADDED Requirements

### Requirement: get-cert command shall display a backend's certificate information
The `oceanctl get cert` command shall query and display the certificate Secret associated with a backend.

#### Scenario: Get certificate for backend in default namespace
- **WHEN** the user runs `oceanctl get cert -b <backend-name>`
- **THEN** the CLI queries the SBC, retrieves the CertSecret name, fetches the Secret, and displays it in table format with columns: Name, Namespace, BoundBackend

#### Scenario: Get certificate for backend in specified namespace
- **WHEN** the user runs `oceanctl get cert -b <backend-name> -n <namespace>`
- **THEN** the CLI performs the query in the specified namespace

#### Scenario: Reject get-cert when backend doesn't exist
- **WHEN** the user runs `oceanctl get cert -b <non-existent-backend>`
- **THEN** the CLI prints "Backend <non-existent-backend> not found"

#### Scenario: Reject get-cert when backend has no certificate
- **WHEN** the user runs `oceanctl get cert -b <backend-name>` and the SBC's CertSecret is empty
- **THEN** the CLI prints "No resource cert in backend <backend-name>"

#### Scenario: Reject get-cert without backend name
- **WHEN** the user runs `oceanctl get cert` without -b flag
- **THEN** the CLI validator rejects the request (ValidateBackend)
