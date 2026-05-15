## ADDED Requirements

### Requirement: delete-backend command shall delete storage backends
The `oceanctl delete backend` command shall delete one or more StorageBackendClaim resources along with their referenced ConfigMap and Secret resources.

#### Scenario: Delete a single backend
- **WHEN** the user runs `oceanctl delete backend <name>`
- **THEN** the CLI queries the SBC, deletes ConfigMap finalizers, deletes ConfigMap + Secret + SBC resources, and prints "backend [<name>] deleted"

#### Scenario: Delete multiple backends
- **WHEN** the user runs `oceanctl delete backend <name1> <name2> -n <namespace>`
- **THEN** the CLI deletes each backend sequentially, printing deletion status for each

#### Scenario: Delete all backends in namespace
- **WHEN** the user runs `oceanctl delete backend -n <namespace> --all`
- **THEN** the CLI queries all SBCs in the namespace and deletes each one

#### Scenario: Delete backend when it doesn't exist
- **WHEN** the user runs `oceanctl delete backend <name>` and the SBC doesn't exist
- **THEN** the CLI prints "Backend <name> not found" and returns without error

#### Scenario: Delete backend with reference resource cleanup
- **WHEN** the user deletes a backend
- **THEN** the CLI deletes ConfigMap finalizers first, then deletes ConfigMap, Secret, SBC, and CertSecret (if configured)

#### Scenario: Reject delete-backend with invalid selector
- **WHEN** the user runs `oceanctl delete backend` without specifying names or --all
- **THEN** the CLI validator rejects the request (ValidateSelector)

#### Scenario: Reject delete-backend with --all and names together
- **WHEN** the user runs `oceanctl delete backend <name> --all`
- **THEN** the CLI validator rejects the request with error "name cannot be provided when a selector is specified"

#### Scenario: Delete backend patches ConfigMap finalizers
- **WHEN** the CLI deletes a backend's ConfigMap
- **THEN** the deleteSbcReferenceResources function first gets the ConfigMap, patches it to remove all finalizers (`{"metadata":{"finalizers":[]}}`), then deletes the ConfigMap, Secret, and SBC

#### Scenario: Delete backend with CertSecret cleanup
- **WHEN** the user deletes a backend that has CertSecret configured
- **THEN** the deleteSbcReferenceResources function includes the CertSecret in the kubectl delete command along with ConfigMap, Secret, and SBC
