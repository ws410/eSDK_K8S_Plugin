
CSI RPC Implementation Map
```text
openspec/specs/csi-rpcs/
├── identity-get-plugin-info/          ✅ Implemented
├── identity-get-plugin-capabilities/  ✅ Implemented
├── identity-probe/                    ✅ Implemented
├── controller-create-volume/          ✅ Implemented (26 scenarios)
├── controller-delete-volume/          ✅ Implemented (6 scenarios)
├── controller-expand-volume/          ✅ Implemented (16 scenarios)
├── controller-publish-volume/         ✅ Implemented (10 scenarios)
├── controller-unpublish-volume/       ✅ Implemented (4 scenarios)
├── controller-validate-volume-capabilities/ ❌ Unimplemented (by design)
├── controller-list-volumes/           ❌ Unimplemented (by design)
├── controller-get-capacity/           ❌ Unimplemented (by design)
├── controller-get-capabilities/       ✅ Implemented (5 capabilities)
├── controller-create-snapshot/        ✅ Implemented (5 scenarios)
├── controller-delete-snapshot/        ✅ Implemented (4 scenarios)
├── controller-list-snapshots/         ❌ Unimplemented (by design)
├── controller-get-volume/             ❌ Unimplemented (by design)
├── node-stage-volume/                 ✅ Implemented (16 scenarios)
├── node-unstage-volume/               ✅ Implemented (6 scenarios)
├── node-publish-volume/               ✅ Implemented (7 scenarios)
├── node-unpublish-volume/             ✅ Implemented (3 scenarios)
├── node-get-info/                     ✅ Implemented (5 scenarios)
├── node-get-capabilities/             ✅ Implemented (3 capabilities)
├── node-get-volume-stats/             ✅ Implemented (10 scenarios)
└── node-expand-volume/                ✅ Implemented (8 scenarios)
```

📊 StoragePlugin 接口方法覆盖率:
```text
方法                    | OceanStor | OceanDisk | DME-A | A-Series | DTree | FusionStorage
────────────────────────────────────────────────────────────────────────────────────
CreateVolume            |    ✅     |    ✅     |  ✅   |    ✅    |  ✅   |     ✅
QueryVolume             |    ✅     |    ✅     |  ✅   |    ✅    |  ✅   |     ✅
DeleteVolume            |    ✅     |    ✅     |  ✅   |    ✅    |  ❌*  |     ✅
ExpandVolume            |    ✅     |    ✅     |  ✅   |    ✅    |  ❌*  |     ✅
AttachVolume            |    ✅     |    ✅     |  ❌†  |    ❌†   |  ✅   |     ✅
DetachVolume            |    ✅     |    ✅     |  ❌†  |    ❌†   |  ✅   |     ✅
CreateSnapshot          |    ✅     |    ❌     |  ❌   |    ❌    |  ❌   |     ✅
DeleteSnapshot          |    ✅     |    ❌     |  ❌   |    ❌    |  ❌   |     ✅
ModifyVolume            |    ✅     |    ❌     |  ❌   |    ❌    |  ❌   |     ✅
DeleteDTreeVolume       |    ❌‡    |    ❌     |  ❌   |    ❌    |  ✅   |     ✅
ExpandDTreeVolume       |    ❌‡    |    ❌     |  ❌   |    ❌    |  ✅   |     ✅
GetSectorSize           |    ✅     |    ✅     |  ✅   |    ✅    |  ✅   |     ✅

```

Key implementation patterns:
- VolumeId format: "backendName.volumeName"
- SnapshotId format: "backendName.parentID.snapshotName"
- NodeId format: JSON-encoded {"HostName": "<hostname>"}
- PublishContext: {"publishInfo": "<json>", "filesystemMode": "<local|HyperMetro>"}
- DTree special handling via constants.IsDtreeStorage()
- Auto-attach fallback when publishInfo missing in NodeStageVolume
- Sector size alignment for all capacity operations