## Purpose

定义 iSCSI 协议连接器的接口规范，用于在 Kubernetes 节点上连接和断开 iSCSI 卷，支持 UltraPath 和 DM-Multipath 多路径配置。

## Requirements

### Requirement: iSCSI 连接器应连接和断开 iSCSI 卷
iSCSI 连接器 SHALL 实现 VolumeConnector 接口，用于在 Kubernetes 节点上连接/断开 iSCSI 卷，支持 UltraPath 和 DM-Multipath。

#### Scenario:使用 DM-Multipath 连接 iSCSI 卷
- **WHEN** 连接器收到带有 iSCSI 参数（tgtPortals, tgtIQNs, tgtLunWWN）的 ConnectVolume 请求时
- **THEN** 连接器发现到每个门户的 iSCSI 会话，登录到目标端，重新扫描 SCSI 总线，按 WWN 发现多路径设备，配置 DM-Multipath，并返回设备路径

#### Scenario:使用 UltraPath 连接 iSCSI 卷
- **WHEN** 连接器收到启用 UltraPath 的 ConnectVolume 请求时
- **THEN** 连接器配置华为 UltraPath 多路径而非 DM-Multipath

#### Scenario:断开 iSCSI 卷
- **WHEN** 连接器收到带有设备 WWN 的 DisConnectVolume 请求时
- **THEN** 连接器移除多路径设备，从 iSCSI 目标端注销，并清理 SCSI 会话

#### Scenario:处理 iSCSI 登录失败
- **WHEN** iSCSI 目标端不可达或认证失败时
- **THEN** 连接器返回带有登录失败原因的错误

#### Scenario:处理 CHAP 认证
- **WHEN** iSCSI 目标端需要 CHAP 认证时
- **THEN** 连接器在登录前配置 CHAP 凭据

#### Scenario:连接 iSCSI 卷时的连接详情
- **WHEN** 连接器连接 iSCSI 卷时
- **THEN** 它执行以下操作：
  - Ping 每个 iSCSI 门户以测试主机连通性，过滤掉不可达门户，仅尝试登录可用门户
  - 为每个门户生成一个 goroutine 用于并发发现、登录和设备扫描；每个 goroutine 都有 panic 恢复机制并原子更新共享数据
  - 将 iscsiadm 配置为手动扫描模式以防止自动设备扫描
  - 扫描时使用指数退避：向 /sys/class/scsi_host/hostX/scan 写入 "scan"，以递增间隔等待，重试直到发现设备或超时
  - 如果启用了 HWUltraPath 多路径，则轮询 UltraPath 设备管理器以发现虚拟设备路径
  - 在建立新连接之前调用 clearResidualPath 移除与 WWN 关联的过时设备路径
