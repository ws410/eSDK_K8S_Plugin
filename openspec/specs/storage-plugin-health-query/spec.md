## Purpose

定义 StoragePlugin 的 QueryVolumeHealth 接口规范，用于轻量级卷健康状态查询。该接口避免了 QueryVolume 的冗余查询，仅执行健康判定所需的最小 API 调用。

## Requirements

### Requirement: StoragePlugin 必须提供 QueryVolumeHealth 轻量级健康检查接口
StoragePlugin interface SHALL 新增 `QueryVolumeHealth` 方法，用于轻量级卷健康状态查询。该方法返回 `*VolumeHealth` 结构体，包含 `Healthy`（bool）和 `Message`（string）字段。该方法 SHALL 避免 QueryVolume 的冗余查询，仅执行健康判定所需的最小 API 调用。

#### Scenario: SAN 存储健康检查返回正常
- **WHEN** 调用 QueryVolumeHealth 查询 SAN 存储卷且 LUN 的 HEALTHSTATUS 为 "1"
- **THEN** 返回 VolumeHealth{Healthy: true, Message: ""}

#### Scenario: SAN 存储健康检查返回异常
- **WHEN** 调用 QueryVolumeHealth 查询 SAN 存储卷且 LUN 的 HEALTHSTATUS 不为 "1"
- **THEN** 返回 VolumeHealth{Healthy: false, Message: "HEALTHSTATUS=<实际值>"}

#### Scenario: NAS 存储卷存在
- **WHEN** 调用 QueryVolumeHealth 查询 NAS 存储卷且文件系统存在
- **THEN** 返回 VolumeHealth{Healthy: true, Message: ""}

#### Scenario: NAS 存储卷不存在
- **WHEN** 调用 QueryVolumeHealth 查询 NAS 存储卷且文件系统不存在
- **THEN** 返回 VolumeHealth{Healthy: false, Message: "volume not found"}

#### Scenario: Dtree 存储卷存在
- **WHEN** 调用 QueryVolumeHealth 查询 Dtree 存储卷且目录树存在
- **THEN** 返回 VolumeHealth{Healthy: true, Message: ""}

#### Scenario: Dtree 存储卷不存在
- **WHEN** 调用 QueryVolumeHealth 查询 Dtree 存储卷且目录树不存在
- **THEN** 返回 VolumeHealth{Healthy: false, Message: "volume not found"}

#### Scenario: 存储后端不可达
- **WHEN** 调用 QueryVolumeHealth 且存储后端不可达
- **THEN** 返回 error，VolumeHealth 为 nil

---

### Requirement: QueryVolumeHealth 必须避免冗余 API 调用
QueryVolumeHealth SHALL 仅执行健康判定所需的最小 API 调用。SAN 存储 SHALL 仅查询 LUN 基本信息，不调用 validateManageWorkLoadType。NAS 存储 SHALL 仅查询文件系统存在性，不调用 workload 验证。Dtree 存储 SHALL 仅查询目录树存在性，不调用 BatchGetQuota。

#### Scenario: SAN 健康检查不调用 workload 验证
- **WHEN** 调用 SAN 存储的 QueryVolumeHealth
- **THEN** 不调用 validateManageWorkLoadType 方法

#### Scenario: NAS 健康检查不调用 workload 验证
- **WHEN** 调用 NAS 存储的 QueryVolumeHealth
- **THEN** 不调用 workload 验证相关方法

#### Scenario: Dtree 健康检查不调用 quota 查询
- **WHEN** 调用 Dtree 存储的 QueryVolumeHealth
- **THEN** 不调用 BatchGetQuota 方法
