## ADDED Requirements

### Requirement: version-conversion is NOT IMPLEMENTED
The CRD version conversion webhook is NOT implemented in the current codebase. There is only one API version (v1) of all CRDs (StorageBackendClaim, StorageBackendContent, VolumeModifyClaim, VolumeModifyContent). No Hub(), ConvertTo(), or ConvertFrom() methods exist on any CRD type. No conversion webhook endpoint is registered.

The codebase does contain a v1beta1-to-v1 AdmissionReview protocol adapter (transformV1beta1AdmitFuncToV1AdmitFunc in pkg/webhook/convert.go), but this is NOT a CRD version conversion webhook -- it is an adapter that allows the same admission handler to process both v1beta1 and v1 AdmissionReview request formats.

#### Scenario: Convert SBC between versions (NOT IMPLEMENTED)
- **WHEN** a request is made to convert a StorageBackendClaim from one API version to another
- **THEN** no conversion webhook exists; only one API version (v1) is defined

#### Scenario: Convert SBCT between versions (NOT IMPLEMENTED)
- **WHEN** a request is made to convert a StorageBackendContent between API versions
- **THEN** no conversion webhook exists; only one API version (v1) is defined

#### Scenario: Convert VMC between versions (NOT IMPLEMENTED)
- **WHEN** a request is made to convert a VolumeModifyClaim between API versions
- **THEN** no conversion webhook exists; only one API version (v1) is defined

#### Scenario: Convert VMCt between versions (NOT IMPLEMENTED)
- **WHEN** a request is made to convert a VolumeModifyContent between API versions
- **THEN** no conversion webhook exists; only one API version (v1) is defined

#### Scenario: AdmissionReview protocol version adapter (IMPLEMENTED)
- **WHEN** the admission webhook receives a v1beta1 AdmissionReview request
- **THEN** the transformV1beta1AdmitFuncToV1AdmitFunc function converts the request to v1 format, processes it through the standard v1 handler, and converts the response back to v1beta1 format
