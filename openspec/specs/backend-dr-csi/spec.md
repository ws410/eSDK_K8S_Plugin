## Purpose

定义 DR-CSI StorageBackend gRPC 服务规范，提供后端注册、状态查询、后端列表和注销功能，用于灾难恢复 CSI 协议的后端管理。

## Requirements

### Requirement: DR-CSI StorageBackend service SHALL manage backends
DR-CSI StorageBackend gRPC 服务 SHALL 为灾难恢复 CSI 协议提供后端注册、状态查询和生命周期管理。

#### Scenario: 注册后端
- **WHEN** storage-backend-controller 通过 DR-CSI 调用 RegisterBackend 时
- **THEN** 服务使用其配置（存储类型、参数、凭据）注册后端并初始化存储插件

#### Scenario:查询后端状态
- **WHEN** sidecar 控制器通过 DR-CSI 调用 GetBackendStatus 时
- **THEN** 服务返回后端的在线状态、能力、池容量和设备规格

#### Scenario:查询所有后端
- **WHEN** sidecar 控制器通过 DR-CSI 调用 ListBackends 时
- **THEN** 服务返回所有已注册后端及其状态

#### Scenario:注销后端
- **WHEN** 后端被删除时
- **THEN** 服务注销后端，从存储阵列登出，并释放客户端连接

#### Scenario:处理后端注册失败
- **WHEN** 注册期间 backend.BuildBackend 失败（配置无效、插件初始化失败）时
- **THEN** 服务返回带有构建失败原因的 gRPC 错误，且不将后端添加到缓存

#### Scenario:AddStorageBackend 对重复后端是幂等的
- **WHEN** 对缓存中已存在的后端调用 AddStorageBackend 时
- **THEN** 服务调用 FetchAndRegisterOneBackend，该函数从 Kubernetes 重新获取并更新缓存条目；不返回重复错误

#### Scenario:RemoveStorageBackend 对不存在的后端是幂等的
- **WHEN** 对缓存中不存在的后端调用 RemoveStorageBackend 时
- **THEN** cacheHandler.Delete 是空操作（对缺失的键安全）；不返回错误

#### Scenario:GetBackendStats 跳过离线后端
- **WHEN** 对 Online=false 的后端调用 GetBackendStats 时
- **THEN** 服务返回错误 "GetBackendStats backend: [X] is offline, skip get stats"，而不查询存储阵列

#### Scenario:GetBackendStats 处理空池
- **WHEN** 对零存储池的后端调用 GetBackendStats 时
- **THEN** 服务在响应中返回空池数组；对零池后端没有特殊处理
