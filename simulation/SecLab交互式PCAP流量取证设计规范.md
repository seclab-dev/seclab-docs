# SecLab 协议仿真 PCAP 取证设计规范

本文档定义协议仿真套件基于 Suite Runtime 的交互式 PCAP 取证实现。

## 1. 职责边界

| 组件 | 职责 |
| --- | --- |
| 套件前端 | 发起开始、停止、下载和删除操作，展示实例级抓包状态。 |
| 套件 API | 维护实例状态，调用 Runtime SDK，将有效 PCAP 保存到套件数据卷。 |
| Runtime SDK | 读取 Agent 注入的运行描述和凭据，封装抓包 API 与二进制响应。 |
| Agent | 校验套件实例与 workload 归属，按 workload 的全部已发布端点抓包。 |

主控不保存协议仿真 PCAP。套件只把成品文件写入自身 `/data/pcap/`，Agent 只在抓包会话期间持有缓冲区。

## 2. Agent Suite Runtime API

Runtime API 以 Agent 服务基路径 `/api/v1/agent` 为基准：

```text
POST /suite-runtime/workloads/{workload_id}/captures
POST /suite-runtime/workloads/{workload_id}/captures/{capture_id}/finish
```

开始抓包不接收请求体。Agent 从已登记 workload 中解析发布端点，避免套件自行提交端口而越过所有权校验。

DNS 双传输 workload 的开始响应示例：

```json
{
  "captureId": "pcap-019c...",
  "status": "capturing",
  "endpoints": [
    { "endpointId": "dns-tcp", "protocol": "tcp", "hostPort": 1053 },
    { "endpointId": "dns-udp", "protocol": "udp", "hostPort": 1053 }
  ]
}
```

停止成功直接返回 `application/vnd.tcpdump.pcap` 二进制内容，不使用 Base64 JSON 包装。套件必须流式读取或按字节读取响应，不得按 UTF-8 文本处理。

Agent 从运行令牌恢复 `suiteId` 和 `instanceId`，并校验目标 workload 属于该套件实例。调用方还必须拥有 `captures.manage` 能力；请求参数不能覆盖这些身份信息。

## 3. 多端点抓包

Agent 内部使用 `PcapMuxHub` 复用宿主机抓包能力：

1. 同时处理 IPv4、IPv6 报文和 TCP、UDP 传输。
2. 捕获槽以 `(transport, host_port)` 为键，不能只按数字端口分流。
3. 同一 workload 的全部已发布端点在一个 capture 中启动和停止。
4. DNS 可同时捕获 `1053/TCP` 与 `1053/UDP`，相同数字端口不会相互覆盖。
5. 报文命中任一端点即写入同一个按时间排序的 PCAP 数据流。
6. 单次抓包默认最长 300 秒；超时后 Agent 必须停止继续收集。

抓包目标来自 workload 的实际发布结果，不来自规则包中的 `containerPort`，也不来自前端显示值。这样用户把主机监听端口改为非默认值后，抓包仍与真实端口一致。

## 4. 套件实例状态

`instances` 表维护以下状态：

| 字段 | 说明 |
| --- | --- |
| `pcap_status` | `idle`、`capturing` 或 `ready`。 |
| `pcap_start_time` | 开始抓包时间。 |
| `pcap_capture_id` | Agent 返回的 capture ID。 |
| `pcap_file_path` | `/data/pcap/` 下的成品文件名。 |

状态流转：

```text
idle --开始成功--> capturing --停止且有效--> ready
  ^                       |                    |
  |                       +--空包或失败--------+
  +----------------删除成品--------------------+
```

处理规则：

1. 只有实例处于运行状态且没有活动 capture 时才能开始。
2. Agent 开始成功后记录 `captureId` 并切换为 `capturing`。
3. 停止响应长度小于等于 24 字节时视为只有 PCAP 文件头，不生成成品，状态复位为 `idle`。
4. 有效内容原子写入 `/data/pcap/` 后切换为 `ready`。
5. `DELETE /api/instances/{id}/pcap` 删除成品并复位状态。
6. workload 删除、套件停用或卸载时，Agent 必须终止相关活动 capture，不能遗留后台抓包任务。

## 5. 套件侧 API 与下载安全

协议仿真套件对前端提供：

```text
POST   /api/instances/{id}/pcap/start
POST   /api/instances/{id}/pcap/stop
POST   /api/instances/{id}/pcap/download
DELETE /api/instances/{id}/pcap
```

下载接口根据实例记录定位文件，不接收任意文件路径。实现必须保证解析后的文件仍位于 `/data/pcap/`，并设置 PCAP 内容类型和安全的附件文件名。不得提供按用户输入文件名读取目录的通用接口。

开始、停止、删除属于状态变更操作，应写入协议仿真套件审计日志；面向平台的关键操作按需通过 `operation-logs.write` 上报，抓包内容和凭据不得进入日志。

## 6. 故障与清理

- Agent API 超时或返回错误时，套件保留可诊断错误信息，但不能伪造 `ready` 状态。
- 停止失败时可以保留 `capturing` 供重试；确认 Agent 已无会话后才复位。
- 套件 API 重启后不得凭内存状态伪造抓包结果；持久化的 `captureId` 用于后续停止或显式清理。
- 删除 workload 前先结束 capture；卸载套件实例时按实例所有权清理全部 workload 与 capture。
- PCAP 文件只面向已授权的当前套件用户下载，不通过 Compose 静态目录公开。
