# SecLab 协议仿真 PCAP 取证设计规范

本文档定义协议仿真套件化后的 PCAP 取证实现。Agent 使用 Rust 用户态原始套接字在宿主机侧抓包，不依赖 `tcpdump`。

## 1. 职责边界

| 组件 | 职责 |
| --- | --- |
| 套件前端 | 发起开启、停止、下载、删除 PCAP 操作。 |
| 套件 API | 维护实例 PCAP 状态，调用 Agent 抓包 API，保存 PCAP 文件。 |
| Agent | 按宿主机端口启动和停止抓包，返回 PCAP 字节。 |

PCAP 文件保存在协议仿真套件数据卷内，主控不保存协议仿真 PCAP 文件。

## 2. Agent 抓包 API

协议仿真套件通过 suite workload API 调用：

```text
POST /api/v1/agent/suite-workloads/pcap/start
POST /api/v1/agent/suite-workloads/pcap/stop
```

开启请求：

```json
{
  "suiteId": "seclab.protocol-simulation",
  "suiteInstanceId": "019f...",
  "workloadId": "workload-...",
  "hostPort": 143
}
```

停止响应：

```json
{
  "captureId": "pcap-...",
  "pcapBytesBase64": "..."
}
```

Agent 校验 workload 属于当前 `suiteInstanceId` 后才允许抓包。

## 3. 抓包实现

Agent 内部使用 `PcapMuxHub`：

1. 全局原始套接字监听宿主机网卡。
2. 默认选择非 Docker bridge/veth 网卡。
3. 每个抓包任务按宿主机端口注册捕获槽。
4. 报文按源端口或目的端口分发。
5. 停止抓包时返回完整 PCAP 字节。
6. 默认最大持续时间为 300 秒。

停用或卸载套件时，Agent 按 `suite_instance_id` 停止该套件实例下所有抓包任务。

## 4. 套件状态

`instances` 表记录：

| 字段 | 说明 |
| --- | --- |
| `pcap_status` | `idle`、`capturing`、`ready`。 |
| `pcap_start_time` | 抓包开始时间戳。 |
| `pcap_capture_id` | Agent 返回的抓包 ID。 |
| `pcap_file_path` | 套件数据卷内保存的 PCAP 文件名。 |

流程：

1. 开启成功后，套件将状态置为 `capturing`。
2. 停止成功后，套件解码 `pcapBytesBase64`。
3. PCAP 字节长度小于等于 24 时视为空包，状态复位为 `idle`，并通过套件 SDK 通知主控展示提醒。
4. 有效 PCAP 写入 `/data/pcap/`，状态置为 `ready`。
5. 删除 PCAP 使用 `DELETE /api/instances/{id}/pcap`，删除文件并复位状态。

## 5. 下载安全

下载 API 为：

```text
GET /api/pcap/download/{file}
```

套件 API 必须拒绝包含 `/`、`\`、`.`、`..` 的文件名，只允许读取套件数据目录下的 `pcap/` 子目录。
