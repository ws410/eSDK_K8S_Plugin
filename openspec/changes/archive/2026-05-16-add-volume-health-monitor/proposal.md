## Why

Kubernetes CSI 外部健康检查器（External Health Monitor）依赖 CSI 驱动的 `ControllerGetVolume` 接口来获取存储卷的健康状态。当前驱动未实现该接口，导致无法向 Kubernetes 报告卷的健康状况，运维人员无法及时发现存储卷异常，影响集群的可靠性和可观测性。

## What Changes

- 实现 `ControllerGetVolume` RPC 接口，查询存储卷健康状态并返回
- 修改 `ControllerGetCapabilities` 接口，新增 `GET_VOLUME` 能力通告
- 新增 volume-monitor sidecar 部署支持，作为可选配置，默认不开启

## Capabilities

### New Capabilities
- `csi-controller-get-volume`: 实现 ControllerGetVolume RPC，返回存储卷的健康状态。支持 SAN 存储（基于健康状态字段判断）、NAS 和 Dtree 存储（基于卷是否存在判断）
- `storage-plugin-health-query`: 新增 StoragePlugin.QueryVolumeHealth 轻量级接口，避免 QueryVolume 的冗余查询。返回 VolumeHealth{Healthy, Message} 结构体

### Modified Capabilities
- `csi-controller-get-capabilities`: ControllerGetCapabilities 接口需新增 GET_VOLUME 能力返回

## Impact

- `pkg/csi/controller/server.go`: 新增 ControllerGetVolume 实现，修改 ControllerGetCapabilities 返回
- `pkg/monitor/`: 新增 volume-monitor sidecar 相关代码
- Helm charts / 部署配置: 新增 volume-monitor sidecar 可选部署配置
- 存储后端适配层: SAN、NAS、Dtree 存储插件需支持健康状态查询
