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
