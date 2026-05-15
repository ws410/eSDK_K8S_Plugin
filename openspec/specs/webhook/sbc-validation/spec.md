## ADDED Requirements

### Requirement: sbc-validation shall validate StorageBackendClaim on create and update
The admission webhook shall validate StorageBackendClaim resources to ensure they meet the required schema and business rules before being persisted. The webhook path is "/storagebackendclaim" with failure policy "Fail". No mutation operations are performed -- the webhook is purely validating.

#### Scenario: Validate SBC with required Provider field
- **WHEN** a user creates an SBC without the Provider field
- **THEN** the validateCommonClaim function rejects the request with error "Provider in StorageBackendClaim [%s] can not be empty"

#### Scenario: Validate SBC ConfigMapMeta format
- **WHEN** a user creates an SBC with ConfigMapMeta not in "<namespace>/<name>" format
- **THEN** the validateCommon function rejects the request; the format is validated indirectly by SplitMetaNamespaceKey during backend.GetStorageBackendInfo(), which returns an error "split configmap meta %s namespace failed"

#### Scenario: Validate SBC SecretMeta format
- **WHEN** a user creates an SBC with SecretMeta not in "<namespace>/<name>" format
- **THEN** the validateCommon function rejects the request; the format is validated indirectly by SplitMetaNamespaceKey during backend.GetStorageBackendInfo(), which returns an error "split secret meta %s namespace failed"

#### Scenario: Validate SBC ConfigMapMeta is not empty
- **WHEN** a user creates an SBC with empty ConfigMapMeta
- **THEN** the validateCommonClaim function rejects the request with error "StorageBackendClaim %s's configmap [%s] is empty"

#### Scenario: Validate SBC SecretMeta is not empty
- **WHEN** a user creates an SBC with empty SecretMeta
- **THEN** the validateCommonClaim function rejects the request with error "StorageBackendClaim %s's secret [%s] is empty"

#### Scenario: Validate SBC update doesn't change immutable fields
- **WHEN** a user updates an SBC's Provider field
- **THEN** the validateUpdate function rejects the request with error "[provider] is forbidden changed with StorageBackendClaim %s"; if only Spec and Annotations are unchanged, the update is allowed without further validation

#### Scenario: Validate SBC with valid configuration
- **WHEN** a user creates an SBC with all required fields and valid formats
- **THEN** the webhook performs full backend validation: retrieves ConfigMap and Secret, constructs the Backend object, validates storage type and plugin existence, validates parameters, and calls Plugin.Validate() which performs a login test to the storage array; if all pass, the request is allowed

#### Scenario: Validate SBC MaxClientThreads range (NOT IMPLEMENTED)
- **WHEN** a user creates or updates an SBC with MaxClientThreads outside the valid range
- **THEN** the webhook does NOT validate MaxClientThreads range; this validation is NOT implemented in the current codebase

#### Scenario: Validate SBC Parameters format (NOT IMPLEMENTED)
- **WHEN** a user creates an SBC with Parameters that contain invalid key-value format
- **THEN** the webhook does NOT validate Parameters format; this validation is NOT implemented in the current codebase

#### Scenario: Validate SBC UseCert and CertSecret consistency (NOT IMPLEMENTED)
- **WHEN** a user creates an SBC with UseCert=true but CertSecret is empty
- **THEN** the webhook does NOT validate UseCert/CertSecret consistency; this validation is NOT implemented in the current codebase

#### Scenario: Validate SBC delete with finalizers
- **WHEN** a user deletes an SBC that has finalizers other than ClaimBoundFinalizer
- **THEN** the validateDelete function rejects the request with error "forbid delete StorageBackendClaim %s, there are some finalizers [%v]"; only the ClaimBoundFinalizer ("storagebackend.xuanwu.huawei.io/storagebackendclaim-bound-protection") is permitted
