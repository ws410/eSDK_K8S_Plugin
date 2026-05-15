## ADDED Requirements

### Requirement: DTree plugin shall manage DTree quota volumes
The DTree plugin (OceanstorDTreePlugin) shall implement the StoragePlugin interface for oceanstor-dtree storage type, managing directory tree quotas on OceanStor NAS systems. It extends the OceanstorPlugin with DTree-specific operations.

#### Scenario: Initialize DTree plugin
- **WHEN** the plugin is initialized
- **THEN** it extracts parentname from parameters (optional, must be string if provided), verifies protocol and portals, validates and creates NfsAutoAuthClient from parameters, and calls the parent OceanstorPlugin init

#### Scenario: Create DTree volume
- **WHEN** CreateVolume is called with volumeType=dtree
- **THEN** the plugin resolves the volume name from PV name or parameters (for Dorado V6/V7 only), determines the valid parentname from StorageClass parameter or backend config, sets vstoreId and parentname in parameters, creates the DTree quota, and sets the DTreeParentName on the volume object

#### Scenario: Query DTree volume
- **WHEN** QueryVolume is called for a DTree volume
- **THEN** the plugin determines the valid parentname from parameters or backend config, and queries the DTree by name, parentName, and vStoreId

#### Scenario: Delete DTree volume
- **WHEN** DeleteDTreeVolume is called
- **THEN** the plugin removes the DTree quota using the volume name, parent name, and vStoreId

#### Scenario: Expand DTree volume
- **WHEN** ExpandDTreeVolume is called
- **THEN** the plugin converts the spaceHardQuota from sector size to bytes using TransK8SCapacity, expands the DTree quota on the array, and returns NodeExpansionRequired=false

#### Scenario: Attach DTree volume with auto auth client
- **WHEN** AttachVolume is called for a DTree volume
- **THEN** the plugin returns the DTreeParentName in the mapping info; if nfsAutoAuthClient is enabled, it filters node IPs by CIDRs, and calls AutoManageAuthClient with ReadWrite access on the DTree

#### Scenario: Detach DTree volume with auto auth client cleanup
- **WHEN** DetachVolume is called for a DTree volume
- **THEN** if nfsAutoAuthClient is enabled, the plugin filters node IPs by CIDRs, calls AutoManageAuthClient with NoAccess, and if IOIsolation is true, checks all clients status before returning

#### Scenario: Reject standard DeleteVolume on DTree
- **WHEN** DeleteVolume (non-DTree variant) is called
- **THEN** the plugin returns error "not implement" (use DeleteDTreeVolume instead)

#### Scenario: Reject standard ExpandVolume on DTree
- **WHEN** ExpandVolume (non-DTree variant) is called
- **THEN** the plugin returns error "not implement" (use ExpandDTreeVolume instead)

#### Scenario: Reject snapshot operations on DTree
- **WHEN** CreateSnapshot is called
- **THEN** the plugin returns error "oceanstor-dtree not support snapshot feature"

#### Scenario: Reject DeleteSnapshot on DTree
- **WHEN** DeleteSnapshot is called
- **THEN** the plugin returns error "not implement"

#### Scenario: DTree disables advanced capabilities
- **WHEN** UpdateBackendCapabilities is called
- **THEN** the plugin inherits from OceanstorPlugin but sets SupportMetro=false, SupportMetroNAS=false, SupportReplication=false, SupportClone=false, SupportApplicationType=false, SupportQoS=false; updates SupportThin for Dorado and updates NFS4 capability from NFS service settings

#### Scenario: DTree pool capacities are zero
- **WHEN** UpdatePoolCapabilities is called
- **THEN** the plugin returns zero capacities for all pool names (DTree uses quota-based capacity, not pool-based)

#### Scenario: Reject ModifyVolume on DTree
- **WHEN** ModifyVolume is called
- **THEN** the plugin returns error "not implement"

#### Scenario: Validate DTree parameters
- **WHEN** Validate is called
- **THEN** the plugin verifies config parameters, validates DTree-specific params (protocol, portals, parentname), creates a test client, performs ValidateLogin, and logs out

#### Scenario: Create DTree volume with Dorado V6/V7 volume name template
- **WHEN** CreateVolume is called for a DTree volume on Dorado V6/V7 storage
- **THEN** the plugin calls getVolumeNameFromPVNameOrParameters to resolve the volume name from the PV name template (validating it contains {{.PVCNamespace}} and {{.PVCName}}, executing with metadata and appending "-{{.PVCUid}}")

#### Scenario: Create DTree volume with parentname mismatch rejection
- **WHEN** CreateVolume is called with both StorageClass parentname and backend parentname set to different values
- **THEN** the getValidParentname function returns an error indicating the parentname values do not match

#### Scenario: Attach DTree volume with parentName resolution priority
- **WHEN** AttachVolume is called for a DTree volume
- **THEN** the attachDTreeVolume function first checks volumeContext[DTreeParentKey]; if found, returns it; otherwise returns empty map; the plugin then falls back to p.parentName if the result is empty

#### Scenario: Reject Attach DTree volume when parentName is missing
- **WHEN** AttachVolume is called for a DTree volume with nfsAutoAuthClient enabled and parentName cannot be determined (not in volumeContext and not in plugin config)
- **THEN** the plugin returns error "failed to get parent name"
