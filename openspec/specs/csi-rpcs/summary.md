
CSI RPC Implementation Map
```text
openspec/specs/csi-rpcs/
├── identity-get-plugin-info/          ✅ Implemented
├── identity-get-plugin-capabilities/  ✅ Implemented
├── identity-probe/                    ✅ Implemented
├── controller-create-volume/          ✅ Implemented
├── controller-delete-volume/          ✅ Implemented
├── controller-expand-volume/          ✅ Implemented
├── controller-publish-volume/         ✅ Implemented
├── controller-unpublish-volume/       ✅ Implemented
├── controller-validate-volume-capabilities/ ❌ Unimplemented
├── controller-list-volumes/           ❌ Unimplemented
├── controller-get-capacity/           ❌ Unimplemented
├── controller-get-capabilities/       ✅ Implemented
├── controller-create-snapshot/        ✅ Implemented
├── controller-delete-snapshot/        ✅ Implemented
├── controller-list-snapshots/         ❌ Unimplemented
├── controller-get-volume/             ❌ Unimplemented
├── node-stage-volume/                 ✅ Implemented
├── node-unstage-volume/               ✅ Implemented
├── node-publish-volume/               ✅ Implemented
├── node-unpublish-volume/             ✅ Implemented
├── node-get-info/                     ✅ Implemented
├── node-get-capabilities/             ✅ Implemented
├── node-get-volume-stats/             ✅ Implemented
└── node-expand-volume/                ✅ Implemented
```