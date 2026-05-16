## ADDED Requirements

### Requirement: backend-cli shall manage storage backends via CLI
The `oceanctl` CLI provides commands for backend lifecycle management: create, get, update, delete backends and certificates, plus log collection and version display.

---

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

#### Scenario: Create backend from YAML with multiple documents and duplicate detection
- **WHEN** the user provides a YAML file with multiple backend documents (separated by "---") containing duplicate backend names
- **THEN** the LoadBackendsFromYaml function detects the duplicate and returns error "the backend X already exists. Please check Y file"

#### Scenario: Create backend with namespace precedence
- **WHEN** the user runs `oceanctl create backend` with namespace specified in CLI flag, YAML file, and default
- **THEN** the namespace precedence is: CLI flag (-n) > YAML file > default (huawei-csi)

#### Scenario: Create backend with storage-specific defaults
- **WHEN** the user creates a backend without specifying maxClientThreads
- **THEN** the CLI sets the default value based on storage type: DME-managed backends (StorageDeviceSN set) get 5, non-DME backends get 30

---

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

---

#### Scenario: Update backend password in default namespace
- **WHEN** the user runs `oceanctl update backend <name> --password`
- **THEN** the CLI queries the SBC, prompts for new password, creates a new Secret with UUID suffix, updates the SBC's secretMeta to point to the new Secret, deletes the old Secret, and prints "backend [<name>] updated"

#### Scenario: Update backend password in specified namespace
- **WHEN** the user runs `oceanctl update backend <name> -n <namespace> --password`
- **THEN** the CLI performs the update in the specified namespace

#### Scenario: Update backend authentication mode to LDAP
- **WHEN** the user runs `oceanctl update backend <name> --password --authenticationMode=ldap`
- **THEN** the CLI converts the authentication mode to scope "1" (LDAP), creates the new Secret with the updated authenticationMode, and updates the SBC

#### Scenario: Update backend authentication mode to local
- **WHEN** the user runs `oceanctl update backend <name> --password --authenticationMode=local`
- **THEN** the CLI converts the authentication mode to scope "0" (local), creates the new Secret, and updates the SBC

#### Scenario: Reject update-backend with non-existent backend
- **WHEN** the user runs `oceanctl update backend <name>` and the SBC doesn't exist
- **THEN** the CLI prints "Backend <name> not found" and returns without error

#### Scenario: Reject update-backend with multiple backend names
- **WHEN** the user runs `oceanctl update backend <name1> <name2>`
- **THEN** the CLI validator rejects the request (ValidateNameIsSingle)

#### Scenario: Reject update-backend with invalid authentication mode
- **WHEN** the user runs `oceanctl update backend <name> --password --authenticationMode=invalid`
- **THEN** the CLI validator rejects the request (ValidateAuthenticationMode)

#### Scenario: Update backend with rollback on failure
- **WHEN** the SBC update fails after the new Secret is created
- **THEN** the CLI restores the old SBC secretMeta by re-applying the old claim YAML, and deletes the newly created Secret

---

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
- **THEN** the CLI deletes ConfigMap finalizers first (patching to remove all finalizers), then deletes ConfigMap, Secret, SBC, and CertSecret (if configured)

#### Scenario: Reject delete-backend with invalid selector
- **WHEN** the user runs `oceanctl delete backend` without specifying names or --all
- **THEN** the CLI validator rejects the request (ValidateSelector)

#### Scenario: Reject delete-backend with --all and names together
- **WHEN** the user runs `oceanctl delete backend <name> --all`
- **THEN** the CLI validator rejects the request with error "name cannot be provided when a selector is specified"

---

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

---

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

---

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

---

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

---

#### Scenario: Display CLI version
- **WHEN** the user runs `oceanctl version`
- **THEN** the CLI outputs the version string (from build-time constants)

---

### Requirement: collect-logs shall collect logs from cluster nodes
The `oceanctl collect logs` command shall collect pod logs from one or all nodes in a specified namespace, display progress, and compress them into a zip archive.

#### Scenario: Collect logs from all nodes in namespace
- **WHEN** the user runs `oceanctl collect logs -n <namespace>`
- **THEN** the CLI validates the namespace exists, queries all pods in the namespace grouped by node, creates local log directories per node, collects logs from each pod concurrently (limited by maxNodeThreads), displays progress for each node, and compresses all logs into a zip file named `<namespace>-<timestamp>-all.zip`

#### Scenario: Collect logs from a specific node
- **WHEN** the user runs `oceanctl collect logs -n <namespace> -N <node-name>`
- **THEN** the CLI validates the node exists, queries pods on that specific node, collects logs, and creates a zip file named `<namespace>-<timestamp>-<node-name>.zip`

#### Scenario: Collect logs with custom thread limit
- **WHEN** the user runs `oceanctl collect logs -n <namespace> -a --threads-max=50`
- **THEN** the CLI limits concurrent node log collection to 50 goroutines

#### Scenario: Reject collect-logs with invalid thread count
- **WHEN** the user provides --threads-max outside the range [1, 1000]
- **THEN** the CLI returns an error "threads-max must in range [1~1000]"

#### Scenario: Reject collect-logs with non-existent namespace
- **WHEN** the user runs `oceanctl collect logs -n <non-existent-namespace>`
- **THEN** the CLI returns an error that the namespace doesn't exist

#### Scenario: Reject collect-logs with non-existent node
- **WHEN** the user runs `oceanctl collect logs -n <namespace> -N <non-existent-node>`
- **THEN** the CLI returns an error that the node doesn't exist

#### Scenario: Collect logs with cleanup on completion
- **WHEN** the log collection completes (success or failure)
- **THEN** the CLI deletes the temporary local log files from /tmp

#### Scenario: Collect logs with symlink skipping
- **WHEN** the CLI encounters symbolic links during log file collection
- **THEN** the CLI skips the symbolic links during compression

#### Scenario: Collect logs handles edge cases
- **WHEN** the CLI collects logs from pods on a node
- **THEN** it handles the following cases:
  - Identifies pod types (CSI, CSM, Xuanwu, Unknown); unknown types generate a warning but collection continues
  - Non-running pods: logs a warning with a manual log path suggestion and continues
  - Collects both current and previous container console logs via `kubectl logs -p`
  - Executes /tmp/collect.sh on the node to gather host information once per node (not per pod)
  - Containers without --log-file-dir argument: logs a warning and skips file log collection (console logs still collected)
  - Final zip includes node file logs, console logs, AND oceanctl logs from /var/log/huawei/oceanctl-log
  - Temporary local log files from /tmp are cleaned up on completion (success or failure)
