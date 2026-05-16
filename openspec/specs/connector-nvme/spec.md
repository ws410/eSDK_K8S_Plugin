## Purpose

定义 NVMe over Fabrics 协议连接器的接口规范，用于通过 RoCE 或 TCP 传输连接和断开 NVMe 卷，支持多路径和会话管理。

## Requirements

### Requirement: NVMe 连接器应连接和断开 NVMe over Fabrics 卷
NVMe 连接器 SHALL 实现 VolumeConnector 接口，用于在 Kubernetes 节点上通过 RoCE 或 TCP 连接/断开 NVMe 卷。

#### Scenario:通过 RoCE 连接 NVMe 卷
- **WHEN** 连接器收到带有 NVMe-RoCE 参数（tgtPortals, tgtLunGuid）的 ConnectVolume 请求时
- **THEN** 连接器从连接属性中提取 tgtLunGuid，使用 NVMe 驱动类型和 tryConnectVolume 函数调用 ConnectVolumeCommon，该函数发现门户上的 NVMe 子系统，连接到 NVMe 目标端，按 GUID 发现命名空间，配置多路径，并返回设备路径

#### Scenario:通过 TCP 连接 NVMe 卷
- **WHEN** 连接器收到带有 NVMe-TCP 参数的 ConnectVolume 请求时
- **THEN** 连接器通过 TCP 传输而非 RoCE 进行连接

#### Scenario:断开 NVMe 卷
- **WHEN** 连接器收到带有设备 GUID 的 DisConnectVolume 请求时
- **THEN** 连接器使用 NVMe 驱动类型和 tryDisConnectVolume 函数调用 DisConnectVolumeCommon，该函数断开与 NVMe 目标端的连接并清理命名空间

#### Scenario:处理未找到 NVMe 启动器
- **WHEN** 节点未配置 NVMe hostnqn（/etc/nvme/hostnqn）时
- **THEN** 连接器返回错误，指示不存在 NVMe 启动器

#### Scenario:拒绝不带 tgtLunGuid 的 ConnectVolume
- **WHEN** 连接器收到连接属性中不包含 tgtLunGuid 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "key tgtLunGuid does not exist in connection properties"

#### Scenario:连接 NVMe 卷时的连接详情
- **WHEN** 连接器连接 NVMe 卷（RoCE 或 TCP）时
- **THEN** 它执行以下操作：
  - 验证 nvme-cli 版本 >= 1.9；如果版本过低则返回错误
  - Ping 每个门户以测试主机连通性，过滤出可用门户（availablePortals）；如果所有门户均不可达则返回错误
  - 将协议映射为传输类型：ProtocolRoce/ProtocolRoceNVMe -> "rdma"，ProtocolTCPNVMe -> "tcp"
  - 运行 `nvme list-subsys -o json` 检查现有连接会话，避免重复连接
  - 如果启用了 HWUltraPathNVMe 多路径，则使用 15 秒超时轮询 `upadmin_plus` 以发现虚拟设备路径
  - 如果启用了 NVMeNative 多路径，则检查 NVMe 多路径是否已启用，使用 findDiskOfNativePath 发现原生多路径设备，并调用 waitAllPathOnline

#### Scenario:断开 NVMe 卷时的会话清理
- **WHEN** 连接器断开 NVMe 卷时
- **THEN** disconnectSessions 函数检查每个会话端口的多个控制器；如果仅剩 1 个控制器，则运行 `nvme disconnect -d <port>`；对于多路径设备，休眠 3 秒后调用 FlushDMDevice，以 20 秒间隔重试 3 次
