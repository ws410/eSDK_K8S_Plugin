## ADDED Requirements

### Requirement: delete-cert command shall delete a backend's certificate
The `oceanctl delete cert` command shall remove the certificate Secret from a backend and update the SBC to disable certificate usage.

#### Scenario: Delete certificate for backend in default namespace
- **WHEN** the user runs `oceanctl delete cert -b <backend-name>`
- **THEN** the CLI queries the SBC, validates the backend exists and has a CertSecret, updates the SBC's UseCert=false and CertSecret="", deletes the Secret resource, and prints "cert [<secret-name>] deleted"

#### Scenario: Delete certificate for backend in specified namespace
- **WHEN** the user runs `oceanctl delete cert -b <backend-name> -n <namespace>`
- **THEN** the CLI performs the deletion in the specified namespace

#### Scenario: Reject delete-cert when backend doesn't exist
- **WHEN** the user runs `oceanctl delete cert -b <non-existent-backend>`
- **THEN** the CLI prints "Backend <non-existent-backend> not found"

#### Scenario: Reject delete-cert when backend has no certificate
- **WHEN** the user runs `oceanctl delete cert -b <backend-name>` and the SBC's CertSecret is empty
- **THEN** the CLI prints "No resource cert in backend <backend-name>"

#### Scenario: Reject delete-cert without backend name
- **WHEN** the user runs `oceanctl delete cert` without -b flag
- **THEN** the CLI validator rejects the request (ValidateBackend)

#### Scenario: Delete cert with rollback on failure
- **WHEN** the SBC update fails after the Secret is deleted
- **THEN** the CLI restores the SBC's UseCert=true and CertSecret, and recreates the Secret (if original cert data is available)
