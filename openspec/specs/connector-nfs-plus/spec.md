## Purpose

定义 NFS+ 协议连接器的接口规范，用于挂载和卸载增强型 NFS 共享，支持多门户连接和基于标签的访问控制机制。

## Requirements

### Requirement: NFS+ 连接器应挂载和卸载增强型 NFS 共享
NFS+ 连接器 SHALL 实现 VolumeConnector 接口，用于挂载/卸载 NFS+ 文件系统共享，支持多门户和基于标签的访问控制。

#### Scenario:使用多门户挂载 NFS+ 共享
- **WHEN** 连接器收到带有 NFS+ 参数（sourcePath, targetPath, portals=[ip1, ip2, ...], mountFlags）的 ConnectVolume 请求时
- **THEN** 连接器解析连接信息，使用 "~" 连接门户作为 remoteaddrs，构造挂载命令 `-t nfs -o remoteaddrs=<portals~separated>,<mountFlags>`，如需则创建目标目录（权限 0750），检查现有挂载，并执行挂载命令

#### Scenario:卸载 NFS+ 共享
- **WHEN** 连接器收到带有 targetPath 的 DisConnectVolume 请求时
- **THEN** 连接器检查目标路径是否存在；如果不存在则返回成功；否则执行 umount，容忍 "not mounted" 或 "not found" 错误，并使用 RemoveAll 删除目标目录

#### Scenario:拒绝缺少门户的挂载
- **WHEN** 连接器收到不带门户或门户为空的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "there are no portals in the connection info"

#### Scenario:拒绝缺少源路径的挂载
- **WHEN** 连接器收到不带 sourcePath 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "there are no source path in the connection info"

#### Scenario:拒绝缺少目标路径的挂载
- **WHEN** 连接器收到不带 targetPath 的 ConnectVolume 请求时
- **THEN** 连接器返回错误 "there are no target path in the connection info"

#### Scenario:挂载 NFS+ 共享时的连接详情
- **WHEN** 连接器挂载 NFS+ 共享时
- **THEN** 它执行以下操作：
  - 验证所有门户要么全是 IP 要么全是域名（不能混合格式）
  - 使用 "~" 连接门户作为 remoteaddrs，构造挂载命令 `-t nfs -o remoteaddrs=<portals~separated>,<mountFlags>`
  - 对于单门户，仍然构造 remoteaddrs 选项（不需要 "~" 分隔符）
  - 如需则创建目标目录（权限 0750）
  - 检查现有挂载：如果源匹配请求的 sourcePath，或目录基名称匹配，或 ContainSourceDevice 确认是同一设备，则返回成功（幂等）；否则返回关于冲突挂载的错误
