# SecLab Dev 文档

`seclab-docs` 是 SecLab Dev 组织级文档源，用于维护跨仓库共享的架构、规范、运行、套件、仿真和 API 治理文档。

业务仓库通过本地软链接接入：

```text
linked-docs -> ../seclab-docs
```

`linked-docs` 不提交到业务仓库 Git，也不作为构建、测试或运行时依赖。

## 文档目录

| 目录 | 内容 |
| --- | --- |
| `architecture/` | 系统架构、Crate 边界、节点管理和计划任务设计。 |
| `standards/` | 主题 Token、运行环境、部署目录、认证、日志和 WebSocket 规范。 |
| `operations/` | 运行时协议、跨 NAT 回连和在线升级规范。 |
| `simulation/` | 协议仿真、PCAP 取证、规则库升级和规则库契约。 |
| `suites/` | Compose 套件中心、套件开发规范和套件 SDK 设计规范。 |
| `api/` | API 设计规范、契约、治理规则和可视化工作台。 |

## API 工作台

```bash
pnpm -C api install
pnpm -C api dev
```

## 维护规则

1. 文档内容使用中文。
2. 文档描述保持正式、简洁、正向。
3. 跨仓库共享规范只在本仓库维护。
4. 业务仓库只保留本仓库文档的软链接入口。
