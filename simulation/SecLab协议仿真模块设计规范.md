# SecLab 协议仿真套件设计规范

本文档定义协议仿真从主控内置模块拆分为 Compose 套件后的当前实现。协议仿真的 UI、业务 API、规则库、实例状态、审计日志和 PCAP 文件由套件维护；主控只提供套件生命周期、入口代理、节点上下文和 SDK 能力。

## 1. 运行边界

协议仿真由三个运行边界组成：

| 边界 | 职责 |
| --- | --- |
| `seclab` 主控 | 导入、安装、启停、卸载套件；代理套件 Web 入口；同步主题、语言、通知和导航能力。 |
| `protocol-simulation` 套件 API/UI | 管理规则、实例、审计日志、PCAP 文件；调用 Agent suite workload API。 |
| `protocol-simulation-engine` workload | 运行具体协议仿真服务，接收规则配置并向套件 API 上报事件。 |
| `seclab-agent` | 在目标节点创建、停止、删除 workload 容器；提供宿主机端口 PCAP 抓包能力。 |

主控不再保留协议仿真专用前端、`/api/v1/simulation/*` 路由、`sim_*` 表和 Agent 内置协议运行器。协议仿真以 `seclab.protocol-simulation` 套件交付。

## 2. 镜像与仓库

协议仿真套件源码仓库为 `seclab-suite-protocol-simulation`，包含两个独立发布的镜像：

| 镜像 | 来源 | 说明 |
| --- | --- | --- |
| `guowenju/seclab-protocol-simulation:<version>` | `crates/protocol-simulation` | 套件 API/UI 服务。 |
| `guowenju/seclab-protocol-simulation-engine:<version>` | `crates/protocol-simulation-engine` | Agent 拉起的规则 workload 容器。 |

套件交付仓库 `seclab-suites` 保存 `suite.yaml`、`compose.yaml`、图标、README 和 CHANGELOG 快照。`suite.yaml.metadata.version` 是套件版本唯一来源；任一镜像 tag 变化时必须同步更新套件版本。

## 3. 数据模型

套件 API 使用套件私有 SQLite 数据库，数据位于套件数据卷内。

| 表 | 说明 |
| --- | --- |
| `rules` | 协议仿真规则。导入规则包时写入包规则，界面创建时写入自定义规则。 |
| `rule_packages` | 当前导入的规则包元数据。 |
| `instances` | 已部署实例状态、workload ID、PCAP 状态和 PCAP 文件路径。 |
| `audit_logs` | engine 上报的交互审计事件。 |

实例下线等同销毁。套件 API 调用 Agent 停止 workload 后删除实例记录；停用或卸载套件时，Agent 会按 `suite_instance_id` 清理仍存活的 workload 容器，套件重新启用后会按 Agent workload 列表校准旧实例状态。

## 4. 套件 API

套件 Web 前端通过主控代理访问套件 API。套件内部 API 使用相对路径，不依赖主控仿真路由。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/health` | 健康检查。 |
| `GET` | `/api/rules` | 查询规则。 |
| `POST` | `/api/rules` | 创建自定义规则。 |
| `DELETE` | `/api/rules/{id}` | 删除规则。 |
| `GET` | `/api/rule-package/current` | 查询当前规则包。 |
| `POST` | `/api/rule-package/import` | 导入 `.slrp` 规则包。 |
| `GET` | `/api/instances` | 查询实例，并与 Agent workload 状态校准。 |
| `POST` | `/api/instances/deploy` | 按规则和端口部署实例。 |
| `POST` | `/api/instances/{id}/undeploy` | 下线实例并删除 workload 容器。 |
| `POST` | `/api/instances/{id}/pcap/start` | 开启实例 PCAP 取证。 |
| `POST` | `/api/instances/{id}/pcap/stop` | 停止取证并保存 PCAP。 |
| `DELETE` | `/api/instances/{id}/pcap` | 删除已保存 PCAP 或停止进行中的取证。 |
| `GET` | `/api/pcap/download/{file}` | 下载 PCAP 文件。 |
| `GET` | `/api/logs` | 查询审计日志。 |
| `POST` | `/internal/events` | engine workload 上报事件。 |

## 5. Workload 编排

套件 API 不挂载 Docker Socket，不直接操作 Docker。部署实例时，套件 API 调用 Agent suite workload API：

```text
POST   /api/v1/agent/suite-workloads/start
POST   /api/v1/agent/suite-workloads/stop
GET    /api/v1/agent/suite-workloads/list
GET    /api/v1/agent/suite-workloads/{workload_id}
DELETE /api/v1/agent/suite-workloads/{workload_id}
POST   /api/v1/agent/suite-workloads/pcap/start
POST   /api/v1/agent/suite-workloads/pcap/stop
```

启动请求包含 `suiteId`、`suiteInstanceId`、`workloadKind`、`workloadName`、镜像、端口映射、环境变量、规则配置和资源限制。协议仿真使用 `workloadKind=simulation-rule`，`workloadName` 使用规则 ID，容器名称形如：

```text
seclab-sim-rule-427001-f3dc45e75209
```

Agent 创建 workload 时强制注入 labels：

```text
seclab.workload_type=suite-workload
seclab.suite_id=<suite_id>
seclab.suite_instance_id=<suite_instance_id>
seclab.workload_id=<workload_id>
seclab.workload_kind=simulation-rule
seclab.workload_name=<rule_id>
```

停用和卸载套件前，Agent 按 `suite_instance_id` 停止 PCAP 任务并删除对应 workload 容器，避免孤儿容器。

## 6. Agent 通信

本地节点使用 Agent UDS；子节点使用 Agent HTTP/mTLS。套件 API 根据运行环境选择：

| 场景 | 连接方式 |
| --- | --- |
| 本地节点 | UDS，URL 形态为 `http://local/...`。 |
| 子节点 | Agent HTTPS/mTLS，证书目录由套件运行环境注入。 |

套件只能使用为当前 `suite_instance_id` 下发的实例级凭据。规则 workload 容器不持有 Agent 凭据，也不接触 Docker Socket。

## 7. 规则包

规则包后缀为 `.slrp`，载荷为 gzip tar。当前套件 API 要求包内包含：

```text
rules.bin
rules.bin.sig
```

`rules.bin` 使用 Protobuf 序列化，包含规则包 manifest 和规则列表。套件 API 当前解析 `rules.bin` 并校验包结构、规则数量、协议类型和规则配置 JSON；`rules.bin.sig` 作为包结构必需文件保留。

导入后，规则 ID 转换为 `sim-rule-<id>`，规则英文名、分类、CVE、描述和协议行为写入 `config_json`，供前端详情、部署和浏览器预览使用。

## 8. 协议能力

engine workload 当前支持：

| 协议 | 默认端口 |
| --- | --- |
| HTTP | 80 |
| Redis | 6379 |
| SMTP | 25 |
| POP3 | 110 |
| IMAP | 143 |
| SSH | 22 |
| FTP | 21 |
| RDP | 3389 |

新增协议时，应在 engine crate 中实现协议运行器，在套件 API 的协议校验中加入协议标识，并更新规则包生成与前端展示逻辑。

## 9. PCAP 取证

PCAP 由 Agent 在宿主机侧抓取，不依赖 `tcpdump`。Agent 监听非 Docker bridge/veth 网卡，按宿主机端口将报文分发到对应抓包槽。

流程：

1. 前端调用套件 API 开启取证。
2. 套件 API 调 Agent `/pcap/start`，传入 `suiteInstanceId`、`workloadId` 和宿主机端口。
3. Agent 返回 `captureId`，套件将实例 `pcap_status` 置为 `capturing`。
4. 停止取证时，Agent 返回 base64 PCAP 字节。
5. 套件 API 将 PCAP 写入自身数据卷，并把实例 `pcap_status` 置为 `ready`。
6. PCAP 低于最小有效大小时，套件使用 SDK 通知主控展示空包提醒，并复位为 `idle`。

抓包任务默认最长 5 分钟。停用或卸载套件时，Agent 会停止该套件实例下仍在运行的抓包任务。

## 10. 套件前端

前端运行在套件 Web 入口内，通过主控代理加载。前端使用：

- `@seclab-dev/vue` 和 SDL Token。
- `@seclab-dev/suite-sdk` 同步主题、语言和通知能力。
- 套件内受限 iframe 承载规则 HTML 预览，不依赖主控内置应用。

套件前端不显示节点选择。当前套件实例天然处于当前节点上下文。
