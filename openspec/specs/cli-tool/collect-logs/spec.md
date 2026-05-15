## ADDED Requirements

### Requirement: collect-logs command shall collect logs from cluster nodes
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

#### Scenario: Group pods by node for log collection
- **WHEN** the CLI collects logs from all nodes
- **THEN** it queries all pods in the namespace, groups them by node name, and creates a separate log directory for each node

#### Scenario: Collect logs concurrently with thread limit
- **WHEN** the CLI collects logs from multiple nodes
- **THEN** it limits concurrent node log collection to maxNodeThreads (default based on configuration) goroutines to prevent overwhelming the API server

#### Scenario: Collect logs identifies pod types
- **WHEN** the CLI collects logs from pods on a node
- **THEN** the NodeLogCollector identifies each pod's type: CSI (huawei-csi pods), CSM (CSI sidecar), Xuanwu (storage-backend-controller/sidecar), or Unknown; unknown pod types generate a warning but collection continues

#### Scenario: Collect logs handles non-running pods
- **WHEN** the CLI encounters a pod that is not in Running state
- **THEN** it logs a warning with a manual log path suggestion for the user to collect logs manually, and continues with other pods

#### Scenario: Collect logs gets console logs for current and previous containers
- **WHEN** the CLI collects logs from a container
- **THEN** it gets both current console logs and previous container logs (using -p flag) via `kubectl logs`, saving them to separate files in the console log directory

#### Scenario: Collect logs gets host information once per node
- **WHEN** the CLI collects logs from the first pod on a node
- **THEN** it executes /tmp/collect.sh on the node to gather host information, copies it to the local log directory, and skips host collection for subsequent pods on the same node

#### Scenario: Collect logs handles container without log-file-dir argument
- **WHEN** the CLI encounters a container that does not have the --log-file-dir argument set
- **THEN** it logs a warning "log-file-dir is not set" and skips file log collection for that container (console logs are still collected)

#### Scenario: Collect logs includes oceanctl logs in zip
- **WHEN** the CLI compresses collected logs
- **THEN** the zip file includes node file logs, console logs, AND oceanctl logs from /var/log/huawei/oceanctl-log
