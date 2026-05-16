## Purpose

定义 `oceanctl` CLI 命令规范，用于存储后端的完整生命周期管理，包括后端和证书的创建、查询、更新、删除，以及集群日志收集和版本显示。

## Requirements

### Requirement: backend-cli SHALL manage storage backends via CLI
`oceanctl` CLI SHALL 提供后端生命周期管理命令：创建、获取、更新、删除后端和证书，以及日志收集和版本显示。

---

#### Scenario:从默认命名空间的 YAML 文件创建后端
- **WHEN** 用户运行 `oceanctl create backend -f /path/to/backend.yaml -i yaml` 时
- **THEN** CLI 从 YAML 文件加载后端，验证后端名称（DNS 格式：小写字母数字或 '-'，最多 63 个字符），将命名空间设置为默认值（huawei-csi），将 provisioner 设置为默认值（csi.huawei.com），根据存储类型设置 maxClientThreads，与已配置的后端合并，显示交互式选择表，并为选定的后端创建 ConfigMap + Secret + StorageBackendClaim

#### Scenario:从指定命名空间的 YAML 文件创建后端
- **WHEN** 用户运行 `oceanctl create backend -f /path/to/backend.yaml -i yaml -n <namespace>` 时
- **THEN** CLI 使用命令行命名空间覆盖文件中的命名空间

#### Scenario:从 JSON 文件创建后端
- **WHEN** 用户运行 `oceanctl create backend -f /path/to/configmap.json -i json` 时
- **THEN** CLI 从 JSON 加载后端，跳过 DNS 名称验证（JSON 的 NotValidateName=true），并正常处理

#### Scenario:使用自定义 provisioner 创建后端
- **WHEN** 用户运行 `oceanctl create backend -f backend.yaml -i yaml --provisioner=custom.provisioner.com` 时
- **THEN** CLI 使用命令行 provisioner 覆盖文件中的 provisioner

#### Scenario:使用 not-validate-name 创建后端
- **WHEN** 用户运行 `oceanctl create backend -f backend.yaml -i yaml --not-validate-name` 时
- **THEN** CLI 跳过 DNS 名称验证，并使用 helper.BuildBackendName 从原始名称自动构建有效的后端名称

#### Scenario:拒绝后端名称为空的 create-backend
- **WHEN** 用户提供的后端配置名称为空时
- **THEN** CLI 返回错误 "backend name cannot be empty"

#### Scenario:拒绝后端名称无效的 create-backend（YAML）
- **WHEN** 用户提供的 YAML 后端名称不符合 DNS 格式（例如包含大写字母、以 '-' 开头/结尾，或超过 63 个字符）时
- **THEN** CLI 返回带有 DNS 格式要求的错误

#### Scenario:拒绝认证模式无效的 create-backend
- **WHEN** 用户提供的后端 authenticationMode 不是 "local" 或 "ldap" 时
- **THEN** CLI 返回验证错误

#### Scenario:拒绝 CIDR 格式无效的 create-backend
- **WHEN** 用户提供的后端 nfsAutoAuthClient=true 且 nfsAutoAuthClientCIDRs 包含无效的 CIDR 字符串时
- **THEN** CLI 返回带有无效 CIDR 的验证错误

#### Scenario:拒绝已配置后端的 create-backend
- **WHEN** 用户选择已配置的后端（Configured=true）时
- **THEN** CLI 打印 "backend [<name>] has been Configured, please select another" 并提示重新选择

#### Scenario:创建后端时失败则回滚
- **WHEN** 用户选择后端且 ConfigMap/Secret/SBC 创建在任何步骤失败时
- **THEN** CLI 通过删除任何部分创建的引用资源（ConfigMap finalizers、ConfigMap、Secret、SBC）进行回滚

#### Scenario:退出交互式 create-backend
- **WHEN** 用户在后端选择提示期间输入 'exit' 时
- **THEN** CLI 退出而不创建任何后端

#### Scenario:从包含多个文档和重复检测的 YAML 创建后端
- **WHEN** 用户提供包含多个后端文档（用 "---" 分隔）且包含重复后端名称的 YAML 文件时
- **THEN** LoadBackendsFromYaml 函数检测到重复并返回错误 "the backend X already exists. Please check Y file"

#### Scenario:创建后端时命名空间优先级
- **WHEN** 用户运行 `oceanctl create backend` 且在 CLI 标志、YAML 文件和默认值中指定了命名空间时
- **THEN** 命名空间优先级为：CLI 标志（-n）> YAML 文件 > 默认值（huawei-csi）

#### Scenario:创建后端时使用存储特定的默认值
- **WHEN** 用户创建后端而未指定 maxClientThreads 时
- **THEN** CLI 根据存储类型设置默认值：DME 管理的后端（设置了 StorageDeviceSN）为 5，非 DME 后端为 30

---

#### Scenario:列出默认命名空间中的所有后端
- **WHEN** 用户运行 `oceanctl get backend` 时
- **THEN** CLI 查询默认命名空间（huawei-csi）中的所有 SBC，并以表格格式显示，列包括：Namespace、Name、Protocol、StorageType、Sn、Status、Online、Url

#### Scenario:列出指定命名空间中的所有后端
- **WHEN** 用户运行 `oceanctl get backend -n <namespace>` 时
- **THEN** CLI 查询指定命名空间中的所有 SBC 并显示它们

#### Scenario:列出指定的后端
- **WHEN** 用户运行 `oceanctl get backend <name1> <name2> -n <namespace>` 时
- **THEN** CLI 查询指定的 SBC 并显示它们，对不存在的后端显示 not-found

#### Scenario:获取后端的宽输出
- **WHEN** 用户运行 `oceanctl get backend -n <namespace> -o wide` 时
- **THEN** CLI 从 SBCT（StorageType、Protocol、Sn）和 ConfigMap（Url）获取额外信息，并显示扩展表格

#### Scenario:获取后端的 JSON 输出
- **WHEN** 用户运行 `oceanctl get backend <name> -o json` 时
- **THEN** CLI 以 JSON 格式输出完整的 SBC 资源

#### Scenario:获取后端的 YAML 输出
- **WHEN** 用户运行 `oceanctl get backend <name> -o yaml` 时
- **THEN** CLI 以 YAML 格式输出完整的 SBC 资源

#### Scenario:获取后端时不存在任何后端
- **WHEN** 用户运行 `oceanctl get backend -n <namespace>` 且没有 SBC 存在时
- **THEN** CLI 打印 "No resource found in <namespace> namespace"

#### Scenario:拒绝输出格式无效的 get-backend
- **WHEN** 用户运行 `oceanctl get backend -o <invalid>` 时
- **THEN** CLI 返回输出格式的验证错误

---

#### Scenario:在默认命名空间中更新后端密码
- **WHEN** 用户运行 `oceanctl update backend <name> --password` 时
- **THEN** CLI 查询 SBC，提示输入新密码，创建带有 UUID 后缀的新 Secret，更新 SBC 的 secretMeta 指向新 Secret，删除旧 Secret，并打印 "backend [<name>] updated"

#### Scenario:在指定命名空间中更新后端密码
- **WHEN** 用户运行 `oceanctl update backend <name> -n <namespace> --password` 时
- **THEN** CLI 在指定命名空间中执行更新

#### Scenario:更新后端认证模式为 LDAP
- **WHEN** 用户运行 `oceanctl update backend <name> --password --authenticationMode=ldap` 时
- **THEN** CLI 将认证模式转换为 scope "1"（LDAP），创建带有更新后的 authenticationMode 的新 Secret，并更新 SBC

#### Scenario:更新后端认证模式为 local
- **WHEN** 用户运行 `oceanctl update backend <name> --password --authenticationMode=local` 时
- **THEN** CLI 将认证模式转换为 scope "0"（local），创建新 Secret，并更新 SBC

#### Scenario:拒绝后端不存在的 update-backend
- **WHEN** 用户运行 `oceanctl update backend <name>` 且 SBC 不存在时
- **THEN** CLI 打印 "Backend <name> not found" 并无错误返回

#### Scenario:拒绝多个后端名称的 update-backend
- **WHEN** 用户运行 `oceanctl update backend <name1> <name2>` 时
- **THEN** CLI 验证器拒绝请求（ValidateNameIsSingle）

#### Scenario:拒绝认证模式无效的 update-backend
- **WHEN** 用户运行 `oceanctl update backend <name> --password --authenticationMode=invalid` 时
- **THEN** CLI 验证器拒绝请求（ValidateAuthenticationMode）

#### Scenario:更新后端时失败则回滚
- **WHEN** SBC 更新在新 Secret 创建之后失败时
- **THEN** CLI 通过重新应用旧的 claim YAML 恢复旧的 SBC secretMeta，并删除新创建的 Secret

---

#### Scenario:删除单个后端
- **WHEN** 用户运行 `oceanctl delete backend <name>` 时
- **THEN** CLI 查询 SBC，删除 ConfigMap finalizers，删除 ConfigMap + Secret + SBC 资源，并打印 "backend [<name>] deleted"

#### Scenario:删除多个后端
- **WHEN** 用户运行 `oceanctl delete backend <name1> <name2> -n <namespace>` 时
- **THEN** CLI 按顺序删除每个后端，打印每个后端的删除状态

#### Scenario:删除命名空间中的所有后端
- **WHEN** 用户运行 `oceanctl delete backend -n <namespace> --all` 时
- **THEN** CLI 查询命名空间中的所有 SBC 并删除每个后端

#### Scenario:删除后端时后端不存在
- **WHEN** 用户运行 `oceanctl delete backend <name>` 且 SBC 不存在时
- **THEN** CLI 打印 "Backend <name> not found" 并无错误返回

#### Scenario:删除后端时清理引用资源
- **WHEN** 用户删除后端时
- **THEN** CLI 首先删除 ConfigMap finalizers（通过 patch 移除所有 finalizer），然后删除 ConfigMap、Secret、SBC 和 CertSecret（如果已配置）

#### Scenario:拒绝选择器无效的 delete-backend
- **WHEN** 用户运行 `oceanctl delete backend` 而未指定名称或 --all 时
- **THEN** CLI 验证器拒绝请求（ValidateSelector）

#### Scenario:拒绝同时使用 --all 和名称的 delete-backend
- **WHEN** 用户运行 `oceanctl delete backend <name> --all` 时
- **THEN** CLI 验证器拒绝请求，错误为 "name cannot be provided when a selector is specified"

---

#### Scenario:在默认命名空间中为后端创建证书
- **WHEN** 用户运行 `oceanctl create cert <name> -f /path/to/cert.crt -b <backend-name>` 时
- **THEN** CLI 加载证书文件内容，按后端名称查询 SBC，验证后端存在，创建带有证书数据（键：tls.crt）的 Secret，更新 SBC 的 UseCert=true 和 CertSecret=<namespace>/<name>，并打印 "cert [<name>] created"

#### Scenario:在指定命名空间中为后端创建证书
- **WHEN** 用户运行 `oceanctl create cert <name> -f /path/to/cert.pem -n <namespace> -b <backend-name>` 时
- **THEN** CLI 在指定命名空间中执行操作

#### Scenario:拒绝后端不存在时的 create-cert
- **WHEN** 用户运行 `oceanctl create cert <name> -f cert.crt -b <non-existent-backend>` 时
- **THEN** CLI 打印 "Backend <non-existent-backend> not found" 并无错误返回

#### Scenario:拒绝后端已有证书时的 create-cert
- **WHEN** 用户运行 `oceanctl create cert <name> -f cert.crt -b <backend-name>` 且 SBC 已设置 CertSecret 时
- **THEN** CLI 返回错误 "a cert already exists on the backend [<backend-name>]"

#### Scenario:拒绝证书名称为空的 create-cert
- **WHEN** 用户运行 `oceanctl create cert` 而未指定名称时
- **THEN** CLI 验证器拒绝请求（ValidateNameIsExist）

#### Scenario:拒绝多个证书名称的 create-cert
- **WHEN** 用户运行 `oceanctl create cert <name1> <name2> -b <backend>` 时
- **THEN** CLI 验证器拒绝请求（ValidateNameIsSingle）

#### Scenario:创建证书时失败则回滚
- **WHEN** Secret 创建之后 SBC 更新失败时
- **THEN** CLI 删除新创建的 Secret

---

#### Scenario:在默认命名空间中获取后端证书
- **WHEN** 用户运行 `oceanctl get cert -b <backend-name>` 时
- **THEN** CLI 查询 SBC，获取 CertSecret 名称，获取 Secret，并以表格格式显示，列包括：Name、Namespace、BoundBackend

#### Scenario:在指定命名空间中获取后端证书
- **WHEN** 用户运行 `oceanctl get cert -b <backend-name> -n <namespace>` 时
- **THEN** CLI 在指定命名空间中执行查询

#### Scenario:拒绝后端不存在时的 get-cert
- **WHEN** 用户运行 `oceanctl get cert -b <non-existent-backend>` 时
- **THEN** CLI 打印 "Backend <non-existent-backend> not found"

#### Scenario:拒绝后端没有证书时的 get-cert
- **WHEN** 用户运行 `oceanctl get cert -b <backend-name>` 且 SBC 的 CertSecret 为空时
- **THEN** CLI 打印 "No resource cert in backend <backend-name>"

#### Scenario:拒绝不带后端名称的 get-cert
- **WHEN** 用户运行 `oceanctl get cert` 而没有 -b 标志时
- **THEN** CLI 验证器拒绝请求（ValidateBackend）

---

#### Scenario:在默认命名空间中更新后端证书
- **WHEN** 用户运行 `oceanctl update cert -b <backend-name> -f /path/to/cert.crt` 时
- **THEN** CLI 查询 SBC，验证后端存在且有 CertSecret，加载新证书文件，更新现有 Secret 的 tls.crt 数据，并打印 "cert [<secret-name>] updated"

#### Scenario:在指定命名空间中更新后端证书
- **WHEN** 用户运行 `oceanctl update cert -b <backend-name> -n <namespace> -f /path/to/cert.pem` 时
- **THEN** CLI 在指定命名空间中执行更新

#### Scenario:拒绝后端不存在时的 update-cert
- **WHEN** 用户运行 `oceanctl update cert -b <non-existent-backend> -f cert.crt` 时
- **THEN** CLI 打印 "Backend <non-existent-backend> not found"

#### Scenario:拒绝后端没有证书时的 update-cert
- **WHEN** 用户运行 `oceanctl update cert -b <backend-name> -f cert.crt` 且 SBC 的 CertSecret 为空时
- **THEN** CLI 打印 "No resource cert in backend <backend-name>"

#### Scenario:拒绝不带后端名称的 update-cert
- **WHEN** 用户运行 `oceanctl update cert -f cert.crt` 而没有 -b 标志时
- **THEN** CLI 验证器拒绝请求（ValidateBackend）

#### Scenario:更新证书时失败则回滚
- **WHEN** Secret 更新失败时
- **THEN** CLI 恢复原始 Secret 数据

---

#### Scenario:在默认命名空间中删除后端证书
- **WHEN** 用户运行 `oceanctl delete cert -b <backend-name>` 时
- **THEN** CLI 查询 SBC，验证后端存在且有 CertSecret，更新 SBC 的 UseCert=false 和 CertSecret=""，删除 Secret 资源，并打印 "cert [<secret-name>] deleted"

#### Scenario:在指定命名空间中删除后端证书
- **WHEN** 用户运行 `oceanctl delete cert -b <backend-name> -n <namespace>` 时
- **THEN** CLI 在指定命名空间中执行删除

#### Scenario:拒绝后端不存在时的 delete-cert
- **WHEN** 用户运行 `oceanctl delete cert -b <non-existent-backend>` 时
- **THEN** CLI 打印 "Backend <non-existent-backend> not found"

#### Scenario:拒绝后端没有证书时的 delete-cert
- **WHEN** 用户运行 `oceanctl delete cert -b <backend-name>` 且 SBC 的 CertSecret 为空时
- **THEN** CLI 打印 "No resource cert in backend <backend-name>"

#### Scenario:拒绝不带后端名称的 delete-cert
- **WHEN** 用户运行 `oceanctl delete cert` 而没有 -b 标志时
- **THEN** CLI 验证器拒绝请求（ValidateBackend）

#### Scenario:删除证书时失败则回滚
- **WHEN** Secret 删除之后 SBC 更新失败时
- **THEN** CLI 恢复 SBC 的 UseCert=true 和 CertSecret，并重新创建 Secret（如果有原始证书数据可用）

---

#### Scenario:显示 CLI 版本
- **WHEN** 用户运行 `oceanctl version` 时
- **THEN** CLI 输出版本字符串（来自构建时常量）

---

### Requirement: collect-logs SHALL collect logs from cluster nodes
`oceanctl collect logs` 命令 SHALL 从指定命名空间中的一个或所有节点收集 pod 日志，显示进度，并压缩为 zip 归档文件。

#### Scenario:从命名空间中的所有节点收集日志
- **WHEN** 用户运行 `oceanctl collect logs -n <namespace>` 时
- **THEN** CLI 验证命名空间存在，查询命名空间中按节点分组的所有 pod，为每个节点创建本地日志目录，并发收集每个 pod 的日志（受 maxNodeThreads 限制），显示每个节点的进度，并将所有日志压缩为名为 `<namespace>-<timestamp>-all.zip` 的 zip 文件

#### Scenario:从特定节点收集日志
- **WHEN** 用户运行 `oceanctl collect logs -n <namespace> -N <node-name>` 时
- **THEN** CLI 验证节点存在，查询该特定节点上的 pod，收集日志，并创建名为 `<namespace>-<timestamp>-<node-name>.zip` 的 zip 文件

#### Scenario:使用自定义线程限制收集日志
- **WHEN** 用户运行 `oceanctl collect logs -n <namespace> -a --threads-max=50` 时
- **THEN** CLI 将并发节点日志收集限制为 50 个 goroutine

#### Scenario:拒绝线程数无效的 collect-logs
- **WHEN** 用户提供的 --threads-max 不在 [1, 1000] 范围内时
- **THEN** CLI 返回错误 "threads-max must in range [1~1000]"

#### Scenario:拒绝命名空间不存在的 collect-logs
- **WHEN** 用户运行 `oceanctl collect logs -n <non-existent-namespace>` 时
- **THEN** CLI 返回命名空间不存在的错误

#### Scenario:拒绝节点不存在的 collect-logs
- **WHEN** 用户运行 `oceanctl collect logs -n <namespace> -N <non-existent-node>` 时
- **THEN** CLI 返回节点不存在的错误

#### Scenario:收集日志完成后清理
- **WHEN** 日志收集完成（成功或失败）时
- **THEN** CLI 删除 /tmp 中的临时本地日志文件

#### Scenario:收集日志时跳过符号链接
- **WHEN** CLI 在日志文件收集期间遇到符号链接时
- **THEN** CLI 在压缩期间跳过符号链接

#### Scenario:收集日志处理边界情况
- **WHEN** CLI 从节点上的 pod 收集日志时
- **THEN** 它处理以下情况：
  - 识别 pod 类型（CSI、CSM、Xuanwu、Unknown）；未知类型生成警告但收集继续
  - 非运行中的 pod：记录警告并附带手动日志路径建议，然后继续
  - 通过 `kubectl logs -p` 收集当前和之前的容器控制台日志
  - 在节点上执行 /tmp/collect.sh 以收集主机信息，每个节点仅执行一次（而非每个 pod）
  - 不带 --log-file-dir 参数的容器：记录警告并跳过文件日志收集（仍收集控制台日志）
  - 最终 zip 包含节点文件日志、控制台日志和来自 /var/log/huawei/oceanctl-log 的 oceanctl 日志
  - 完成后（成功或失败）清理来自 /tmp 的临时本地日志文件
