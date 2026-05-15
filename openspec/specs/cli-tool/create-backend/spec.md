## ADDED Requirements

### Requirement: create-backend command shall configure a new storage backend
The `oceanctl create backend` command shall create a StorageBackendClaim by loading backend configuration from a YAML or JSON file, validating it, merging with existing backends, and creating ConfigMap + Secret + SBC resources interactively.

#### Scenario: Create backend from YAML file in default namespace
- **WHEN** the user runs `oceanctl create backend -f /path/to/backend.yaml -i yaml`
- **THEN** the CLI loads backends from the YAML file, validates backend names (DNS format: lowercase alphanumeric or '-', max 63 chars), sets namespace to default (huawei-csi), sets provisioner to default (csi.huawei.com), sets maxClientThreads based on storage type, merges with configured backends, displays an interactive selection table, and creates ConfigMap + Secret + StorageBackendClaim for the selected backend

#### Scenario: Create backend from YAML file in specified namespace
- **WHEN** the user runs `oceanctl create backend -f /path/to/backend.yaml -i yaml -n <namespace>`
- **THEN** the CLI overrides the namespace from the file with the command-line namespace

#### Scenario: Create backend from JSON file
- **WHEN** the user runs `oceanctl create backend -f /path/to/configmap.json -i json`
- **THEN** the CLI loads backends from JSON, skips DNS name validation (NotValidateName=true for JSON), and processes normally

#### Scenario: Create backend with custom provisioner
- **WHEN** the user runs `oceanctl create backend -f backend.yaml -i yaml --provisioner=custom.provisioner.com`
- **THEN** the CLI overrides the provisioner from the file with the command-line provisioner

#### Scenario: Create backend with not-validate-name
- **WHEN** the user runs `oceanctl create backend -f backend.yaml -i yaml --not-validate-name`
- **THEN** the CLI skips DNS name validation and auto-builds a valid backend name from the original name using helper.BuildBackendName

#### Scenario: Reject create-backend with empty backend name
- **WHEN** the user provides a backend configuration with an empty name
- **THEN** the CLI returns an error "backend name cannot be empty"

#### Scenario: Reject create-backend with invalid backend name (YAML)
- **WHEN** the user provides a YAML backend with a name that doesn't match DNS format (e.g., contains uppercase, starts/ends with '-', or exceeds 63 chars)
- **THEN** the CLI returns an error with the DNS format requirements

#### Scenario: Reject create-backend with invalid authentication mode
- **WHEN** the user provides a backend with authenticationMode that is not "local" or "ldap"
- **THEN** the CLI returns a validation error

#### Scenario: Reject create-backend with invalid CIDR format
- **WHEN** the user provides a backend with nfsAutoAuthClient=true and nfsAutoAuthClientCIDRs containing invalid CIDR strings
- **THEN** the CLI returns a validation error with the invalid CIDR

#### Scenario: Reject create-backend with already configured backend
- **WHEN** the user selects a backend that is already configured (Configured=true)
- **THEN** the CLI prints "backend [<name>] has been Configured, please select another" and prompts for another selection

#### Scenario: Create backend with rollback on failure
- **WHEN** the user selects a backend and the ConfigMap/Secret/SBC creation fails at any step
- **THEN** the CLI rolls back by deleting any partially created reference resources (ConfigMap finalizers, ConfigMap, Secret, SBC)

#### Scenario: Exit create-backend interactively
- **WHEN** the user enters 'exit' during the backend selection prompt
- **THEN** the CLI exits without creating any backend
