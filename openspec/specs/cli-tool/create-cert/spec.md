## ADDED Requirements

### Requirement: create-cert command shall create a certificate for a backend
The `oceanctl create cert` command shall create a Kubernetes Secret containing a TLS certificate and update the associated StorageBackendClaim to enable certificate usage.

#### Scenario: Create certificate for backend in default namespace
- **WHEN** the user runs `oceanctl create cert <name> -f /path/to/cert.crt -b <backend-name>`
- **THEN** the CLI loads the certificate file content, queries the SBC by backend name, validates the backend exists, creates a Secret with the certificate data (key: tls.crt), updates the SBC's UseCert=true and CertSecret=<namespace>/<name>, and prints "cert [<name>] created"

#### Scenario: Create certificate for backend in specified namespace
- **WHEN** the user runs `oceanctl create cert <name> -f /path/to/cert.pem -n <namespace> -b <backend-name>`
- **THEN** the CLI performs the operation in the specified namespace

#### Scenario: Reject create-cert when backend doesn't exist
- **WHEN** the user runs `oceanctl create cert <name> -f cert.crt -b <non-existent-backend>`
- **THEN** the CLI prints "Backend <non-existent-backend> not found" and returns without error

#### Scenario: Reject create-cert when backend already has a certificate
- **WHEN** the user runs `oceanctl create cert <name> -f cert.crt -b <backend-name>` and the SBC already has CertSecret set
- **THEN** the CLI returns an error "a cert already exists on the backend [<backend-name>]"

#### Scenario: Reject create-cert with empty cert name
- **WHEN** the user runs `oceanctl create cert` without specifying a name
- **THEN** the CLI validator rejects the request (ValidateNameIsExist)

#### Scenario: Reject create-cert with multiple cert names
- **WHEN** the user runs `oceanctl create cert <name1> <name2> -b <backend>`
- **THEN** the CLI validator rejects the request (ValidateNameIsSingle)

#### Scenario: Create cert with rollback on failure
- **WHEN** the SBC update fails after the Secret is created
- **THEN** the CLI deletes the newly created Secret
