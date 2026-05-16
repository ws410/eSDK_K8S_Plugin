## ADDED Requirements

### Requirement: sbct-validation is NOT IMPLEMENTED
The SBCT (StorageBackendContent) validation webhook is NOT implemented in the current codebase. There is no webhook handler registered for storagebackendcontents resources. No admitStorageBackendContent function exists.

Validation of SBCT fields occurs indirectly through controller-side logic (e.g., SplitMetaNamespaceKey on BackendClaim in backend_register.go), but this is runtime validation, not admission webhook validation.
