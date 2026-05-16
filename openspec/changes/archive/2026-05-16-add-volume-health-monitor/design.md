## Context

当前 CSI 驱动未实现 `ControllerGetVolume` RPC，对所有请求返回 `codes.Unimplemented`。Kubernetes 外部健康检查器（csi-external-health-monitor-controller）依赖此接口获取卷健康状态，用于在 PVC 上设置 `VolumeHealth` 条件。

现有存储后端：
- **SAN 存储**（oceanstor-san, fusionstorage-san）：存储阵列返回 LUN 的 `HEALTHSTATUS` 字段，可用于判断健康状态
- **NAS 存储**（oceanstor-nas, fusionstorage-nas）：通过卷是否存在判断健康状态
- **Dtree 存储**（oceanstor-dtree, fusionstorage-dtree）：目录树无明确健康状态字段，通过卷是否存在判断

约束：
- volume-monitor sidecar 使用社区官方镜像 `registry.k8s.io/sig-storage/csi-external-health-monitor-controller:v0.10.0`
- 默认不开启 volume-monitor sidecar
- 不需要返回发布状态（VolumeCondition 中的 `published` 字段）
- 不复用现有 `QueryVolume` 接口（存在冗余查询），需新增专用健康检查接口

## Goals / Non-Goals

**Goals:**
- 实现 `ControllerGetVolume` RPC，返回卷 ID 和健康状态
- 修改 `ControllerGetCapabilities` 返回 `GET_VOLUME` 能力
- 提供可选的 volume-monitor sidecar 部署配置
- 支持 SAN/NAS/Dtree 三种存储类型的健康状态判定
- 新增轻量级 `QueryVolumeHealth` 接口，避免 QueryVolume 的冗余查询

**Non-Goals:**
- 不实现 Volume 发布状态检测
- 不修改现有卷创建/删除/扩容逻辑
- 不实现 Node 侧的健康检测（已有 NodeGetVolumeStats）
- 不实现 `ListVolumes` 接口

## Decisions

### Decision 1: 新增 QueryVolumeHealth 专用接口

**选择**: 在 `StoragePlugin` interface 中新增 `QueryVolumeHealth` 方法，而非复用 `QueryVolume`。

```go
type VolumeHealth struct {
    Healthy bool   // 卷是否健康
    Message string // 健康状态描述信息（异常时提供原因）
}

QueryVolumeHealth(ctx context.Context, name string, params map[string]interface{}) (*VolumeHealth, error)
```

**QueryVolume 的冗余问题**:
- **SAN**: 查询完整 LUN 对象（50+字段），含 `validateManageWorkLoadType` 额外调用
- **NAS**: 查询完整 filesystem 对象（30+字段），含 workload 验证
- **Dtree**: 调用 **2 次 API**（GetDTreeByName + BatchGetQuota）

**QueryVolumeHealth 的轻量实现**:
- **SAN**: 仅查询 LUN 基本信息，检查 `HEALTHSTATUS` 字段
- **NAS**: 仅查询 filesystem 是否存在
- **Dtree**: 仅查询 DTree 是否存在

**理由**:
- 健康检查是高频调用，需避免不必要的 API 开销
- 专用接口职责单一，便于后续优化和测试
- 与现有 QueryVolume 解耦，互不影响

**替代方案**:
- 复用 QueryVolume：实现简单但性能差，不符合高频健康检查场景
- 创建独立的健康检查服务：增加复杂度，不必要

### Decision 2: 健康状态判定逻辑

**选择**: 按存储类型差异化处理：
- **SAN**: 查询 LUN 的 `HEALTHSTATUS` 字段，值为 `"1"` 表示正常，其他值表示异常，Message 记录具体状态码
- **NAS**: 调用轻量查询，卷存在则 Healthy=true，卷不存在则 Healthy=false
- **Dtree**: 同 NAS 逻辑

**理由**:
- SAN 存储的 LUN 对象有明确的 `HEALTHSTATUS` 字段，可提供更精确的健康判断
- NAS 文件系统虽有 HEALTHSTATUS 字段，但按需求采用卷存在性判定
- Dtree 目录树无健康状态字段，卷存在即正常是合理的判定方式

### Decision 3: Volume-monitor Sidecar 部署方式

**选择**: 在 csi-controller Pod 中以 sidecar 容器形式部署社区官方 sidecar。

**Sidecar 信息**:
- 名称: `csi-external-health-monitor-controller`
- 镜像: `registry.k8s.io/sig-storage/csi-external-health-monitor-controller:v0.10.0`
- 启动参数: `--csiAddress=/csi/csi.sock --monitor-interval=60s`

**Sidecar 职责**:
- 定期调用 CSI Driver 的 `ControllerGetVolume` 接口
- 将健康状态上报到 Kubernetes PVC 的 `VolumeHealth` 条件

**配置方式**:
- `values.yaml` 中新增 `controller.volumeMonitor.enabled` 字段，默认 `false`
- 启用时在 Deployment 中添加 sidecar 容器
- 保持 `ListVolumes` 为 Unimplemented，强制 sidecar 使用 `ControllerGetVolume`

**理由**:
- 使用社区官方 sidecar，无需自研维护
- Sidecar 模式与现有 csi-provisioner/attacher 等 sidecar 保持一致
- 共享 CSI socket，无需额外网络配置
- 可选配置满足用户需求

### Decision 4: 健康状态缓存策略

**选择**: `ControllerGetVolume` 每次调用实时查询存储后端，不做缓存。

**理由**:
- CSI 规范要求实时状态
- 外部健康检查器调用频率可控（默认 1 分钟）
- 避免缓存导致的状态延迟

## Risks / Trade-offs

| Risk | Mitigation |
|------|-----------|
| 存储后端不可用时 ControllerGetVolume 调用超时 | 设置合理的超时时间，返回适当错误码 |
| 大量卷同时查询对存储后端造成压力 | 外部健康检查器有内置限流，sidecar 可配置检查间隔 |
| NAS/Dtree 存储无法检测文件系统内部故障 | 明确文档说明，仅检测卷级别存在性 |
| Sidecar 增加 Pod 资源消耗 | 默认关闭，用户按需启用；sidecar 资源请求设置合理下限 |
