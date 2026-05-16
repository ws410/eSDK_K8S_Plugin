## 1. 新增 QueryVolumeHealth 接口

- [x] 1.1 在 `csi/backend/plugin/plugin.go` 中定义 `VolumeHealth` 结构体（Healthy bool, Message string）
- [x] 1.2 在 `StoragePlugin` interface 中新增 `QueryVolumeHealth` 方法签名
- [x] 1.3 实现 SAN 存储 `QueryVolumeHealth`：查询 LUN 基本信息，检查 HEALTHSTATUS 字段
- [x] 1.4 实现 NAS 存储 `QueryVolumeHealth`：仅查询 filesystem 存在性
- [x] 1.5 实现 Dtree 存储 `QueryVolumeHealth`：仅查询 DTree 存在性

## 2. ControllerGetVolume 接口实现

- [x] 2.1 在 `csi/driver/controller.go` 中实现 `ControllerGetVolume` 方法，校验 volume_id 参数
- [x] 2.2 调用 `QueryVolumeHealth` 获取健康状态并构造 CSI 响应
- [x] 2.3 处理存储后端不可用场景，返回 codes.Internal 错误
- [x] 2.4 添加 ControllerGetVolume 相关日志记录

## 3. ControllerGetCapabilities 接口变更

- [x] 3.1 在 `ControllerGetCapabilities` 返回中新增 `GET_VOLUME` 能力

## 4. 单元测试

- [ ] 4.1 编写 QueryVolumeHealth 单元测试：SAN 存储健康/异常场景
- [ ] 4.2 编写 QueryVolumeHealth 单元测试：NAS 存储存在/不存在场景
- [ ] 4.3 编写 QueryVolumeHealth 单元测试：Dtree 存储存在/不存在场景
- [ ] 4.4 编写 QueryVolumeHealth 单元测试：验证无冗余 API 调用
- [ ] 4.5 编写 ControllerGetVolume 单元测试：健康状态映射到 CSI 响应
- [ ] 4.6 编写 ControllerGetVolume 单元测试：卷 ID 为空场景
- [ ] 4.7 编写 ControllerGetCapabilities 单元测试：验证返回 GET_VOLUME 能力

## 5. Helm 部署配置

- [x] 5.1 在 `values.yaml` 中新增 `controller.volumeMonitor.enabled` 配置项，默认 false
- [x] 5.2 在 `huawei-csi-controller.yaml` 中添加 `csi-external-health-monitor-controller:v0.10.0` sidecar 容器模板（条件渲染）
- [x] 5.3 配置 sidecar 启动参数：`--csiAddress=/csi/csi.sock --monitor-interval=60s`
- [x] 5.4 配置 sidecar 资源请求和限制
- [x] 5.5 配置 sidecar 的 CSI socket 挂载卷
