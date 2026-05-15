## ADDED Requirements

### Requirement: get-backend command shall query and display storage backends
The `oceanctl get backend` command shall query StorageBackendClaim resources and display them in table format (default/wide) or structured format (JSON/YAML).

#### Scenario: List all backends in default namespace
- **WHEN** the user runs `oceanctl get backend`
- **THEN** the CLI queries all SBCs in the default namespace (huawei-csi) and displays them in table format with columns: Namespace, Name, Protocol, StorageType, Sn, Status, Online, Url

#### Scenario: List all backends in specified namespace
- **WHEN** the user runs `oceanctl get backend -n <namespace>`
- **THEN** the CLI queries all SBCs in the specified namespace and displays them

#### Scenario: List specified backends
- **WHEN** the user runs `oceanctl get backend <name1> <name2> -n <namespace>`
- **THEN** the CLI queries the specified SBCs and displays them, showing not-found backends for any that don't exist

#### Scenario: Get backend with wide output
- **WHEN** the user runs `oceanctl get backend -n <namespace> -o wide`
- **THEN** the CLI fetches additional information from SBCT (StorageType, Protocol, Sn) and ConfigMap (Url), and displays the extended table

#### Scenario: Get backend with JSON output
- **WHEN** the user runs `oceanctl get backend <name> -o json`
- **THEN** the CLI outputs the full SBC resource in JSON format

#### Scenario: Get backend with YAML output
- **WHEN** the user runs `oceanctl get backend <name> -o yaml`
- **THEN** the CLI outputs the full SBC resource in YAML format

#### Scenario: Get backend when none exist
- **WHEN** the user runs `oceanctl get backend -n <namespace>` and no SBCs exist
- **THEN** the CLI prints "No resource found in <namespace> namespace"

#### Scenario: Reject get-backend with invalid output format
- **WHEN** the user runs `oceanctl get backend -o <invalid>`
- **THEN** the CLI returns a validation error for the output format
