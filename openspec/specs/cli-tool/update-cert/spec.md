## ADDED Requirements

### Requirement: update-cert command shall update a backend's certificate
The `oceanctl update cert` command shall replace the TLS certificate in the existing Secret associated with a backend.

#### Scenario: Update certificate for backend in default namespace
- **WHEN** the user runs `oceanctl update cert -b <backend-name> -f /path/to/cert.crt`
- **THEN** the CLI queries the SBC, validates the backend exists and has a CertSecret, loads the new certificate file, updates the existing Secret's tls.crt data, and prints "cert [<secret-name>] updated"

#### Scenario: Update certificate for backend in specified namespace
- **WHEN** the user runs `oceanctl update cert -b <backend-name> -n <namespace> -f /path/to/cert.pem`
- **THEN** the CLI performs the update in the specified namespace

#### Scenario: Reject update-cert when backend doesn't exist
- **WHEN** the user runs `oceanctl update cert -b <non-existent-backend> -f cert.crt`
- **THEN** the CLI prints "Backend <non-existent-backend> not found"

#### Scenario: Reject update-cert when backend has no certificate
- **WHEN** the user runs `oceanctl update cert -b <backend-name> -f cert.crt` and the SBC's CertSecret is empty
- **THEN** the CLI prints "No resource cert in backend <backend-name>"

#### Scenario: Reject update-cert without backend name
- **WHEN** the user runs `oceanctl update cert -f cert.crt` without -b flag
- **THEN** the CLI validator rejects the request (ValidateBackend)

#### Scenario: Update cert with rollback on failure
- **WHEN** the Secret update fails
- **THEN** the CLI restores the original Secret data
