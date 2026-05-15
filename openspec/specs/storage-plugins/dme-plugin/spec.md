## ADDED Requirements

### Requirement: DME A-Series plugin shall manage A-Series via DME
The DME plugin (DMEASeriesPlugin) shall implement the StoragePlugin interface for oceanstor-a-series-nas-dme storage type, managing A-Series storage through the DME (Data Management Engine) API. It supports NFS and DTFS protocols.

#### Scenario: Initialize DME A-Series plugin
- **WHEN** the plugin is initialized
- **THEN** it validates protocol is "nfs" or "dtfs", verifies portals (required for NFS with exactly 1 portal; not required for DTFS), verifies storageDeviceSN exists in config, creates a DME client, logs in, sets system info with the device SN, and optionally logs out based on keepLogin flag

#### Scenario: Create volume via DME
- **WHEN** CreateVolume is called
- **THEN** the plugin resolves the volume name from PV name or parameters, converts parameters to CreateDmeVolumeParameter struct, generates the volume model with protocol and sector size, and creates the volume via the DME API

#### Scenario: Query volume via DME
- **WHEN** QueryVolume is called
- **THEN** the plugin queries the volume by name through the DME API and returns volume metadata

#### Scenario: Delete volume via DME
- **WHEN** DeleteVolume is called
- **THEN** the plugin deletes the volume by name and protocol through the DME API

#### Scenario: Expand volume via DME
- **WHEN** ExpandVolume is called
- **THEN** the plugin expands the volume capacity (size * sectorSize) through the DME API and returns NodeExpansionRequired=false

#### Scenario: Set default maxClientThreads for DME
- **WHEN** maxClientThreads is not specified in the backend configuration
- **THEN** the CLI sets it to DMEDefaultMaxClientThreads (different from the standard default)

#### Scenario: DME capabilities are fixed
- **WHEN** UpdateBackendCapabilities is called
- **THEN** the plugin returns fixed capabilities: SupportThin=true, all others (SupportApplicationType, SupportQoS, SupportThick, SupportMetro, SupportReplication, SupportClone, SupportMetroNAS, SupportConsistentSnapshot) = false

#### Scenario: Reject snapshot operations on DME A-Series
- **WHEN** CreateSnapshot or DeleteSnapshot is called
- **THEN** the plugin returns error "oceanstor-a-series-nas-dme storage does not support snapshot feature"

#### Scenario: Reject DTree operations on DME A-Series
- **WHEN** DeleteDTreeVolume or ExpandDTreeVolume is called
- **THEN** the plugin returns error "oceanstor-a-series-nas-dme storage does not support DTree feature"

#### Scenario: Reject ModifyVolume on DME A-Series
- **WHEN** ModifyVolume is called
- **THEN** the plugin returns error "oceanstor-a-series-nas-dme storage does not support volume modify feature"

#### Scenario: SupportQoSParameters always passes for DME
- **WHEN** SupportQoSParameters is called
- **THEN** the plugin returns nil (always passes, no actual validation)

#### Scenario: Update pool capabilities via DME
- **WHEN** UpdatePoolCapabilities is called
- **THEN** the plugin queries HyperScale pools from DME, converts capacities from MB to bytes using DmeCapacityUnitMb, and returns pool capacity maps

#### Scenario: Validate DME parameters
- **WHEN** Validate is called
- **THEN** the plugin verifies parameters, protocol, portals, creates a test client, performs ValidateLogin, and logs out
