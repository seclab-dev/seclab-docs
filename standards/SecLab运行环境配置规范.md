# SecLab 运行环境配置规范

## 1. 目标

本文档定义 SecLab 的环境变量、`.env` 加载规则，以及开发环境和生产环境的路径推导模型。

运行环境分为两类：

1. **开发环境**：运行产物默认写入仓库内 `.seclab/`，避免依赖 root 权限。
2. **生产环境**：运行数据以 `SECLAB_HOME=/opt/seclab` 为根目录，由 systemd、容器编排或部署脚本注入。

二进制入口路径不由本文档定义，生产部署中的 `seclab`、`seclab-agent` 和 `slctl` 安装在 `/usr/local/bin`，并在 `/usr/bin` 创建软链接。

## 2. 解析优先级

运行时配置按以下优先级解析：

```text
进程环境变量 > .env > 代码默认值
```

规则：

1. 进程环境变量用于 CI、systemd、Docker Compose 和生产部署覆盖。
2. `.env` 仅用于本地开发，模板文件为 `.example.env`。
3. 代码默认值必须允许普通用户在开发环境直接启动服务。

## 3. 开发路径模型

开发环境推荐配置：

```env
SECLAB_DATA_DIR=.seclab
SECLAB_DB_DIR=.seclab/database
SECLAB_LOG_DIR=.seclab/logs
SECLAB_CONFIG_DIR=.seclab/config
SECLAB_AGENT_SOCKET=.seclab/seclab-agent.sock
RUST_LOG=info
```

路径推导结果：

```text
.seclab/
├── database/
│   ├── seclab.db
│   └── agent.db
├── logs/
│   ├── seclab/
│   └── agent/
├── config/
│   ├── seclab.toml
│   ├── agent.toml
│   ├── runtime-listen.json
│   └── runtime-listen.key
└── seclab-agent.sock
```

新配置必须使用 `SECLAB_DATA_DIR`。`SECLAB_DEV_HOME` 仅用于历史配置迁移。

## 4. 生产路径模型

生产环境不得依赖仓库 `.env`。默认生产根目录为 `/opt/seclab`：

```env
SECLAB_HOME=/opt/seclab
```

路径推导结果：

```text
/opt/seclab/
├── config/
│   ├── seclab.toml
│   ├── agent.toml
│   ├── runtime-listen.json
│   └── runtime-listen.key
├── database/
│   ├── seclab.db
│   └── agent.db
├── data/
│   └── compose/
│       ├── docker/
│       │   └── <project-name>/compose.yaml
│       └── suite/
│           └── <suite-project-name>/compose.yaml
├── run/
│   └── seclab-agent.sock
└── logs/
    ├── seclab/
    └── agent/
```

数据库、日志、配置或 Socket 需要独立挂载时，使用单项变量覆盖：

```env
SECLAB_DB_DIR=/data/seclab/database
SECLAB_LOG_DIR=/data/seclab/logs
SECLAB_CONFIG_DIR=/opt/seclab/config
SECLAB_AGENT_SOCKET=/opt/seclab/run/seclab-agent.sock
```

权限要求：

- 数据库目录需要读写权限。
- 日志目录需要创建子目录和写入文件权限。
- socket 所在目录需要创建、删除 Unix socket 的权限。

## 5. 变量说明

| 变量 | 用途 |
| --- | --- |
| `SECLAB_HOME` | 生产应用根目录，默认 `/opt/seclab` |
| `SECLAB_DATA_DIR` | 开发产物根目录，未显式设置子目录时用于推导开发路径 |
| `SECLAB_DB_DIR` | SQLite 数据库目录 |
| `SECLAB_LOG_DIR` | 运行日志根目录 |
| `SECLAB_PUBLIC_HOST` | 默认访问主机名或 IP，用于生成节点默认回连地址；首次安装时可由 `install.sh` 写入运行时网络配置 |
| `SECLAB_CONFIG_DIR` | 配置文件目录 |
| `SECLAB_CONFIG` | SecLab 主控配置文件路径，优先于 `SECLAB_CONFIG_DIR/seclab.toml` |
| `SECLAB_AGENT_CONFIG` | Agent 配置文件路径，优先于 `SECLAB_CONFIG_DIR/agent.toml` |
| `SECLAB_RUNTIME_CONFIG` | 主控监听运行时配置路径 |
| `SECLAB_RUNTIME_KEY` | 主控监听运行时配置加密密钥路径 |
| `SECLAB_AGENT_SOCKET` | 本地 Agent Unix socket 路径 |
| `SECLAB_DEV_HOME` | 历史兼容变量，新代码不推荐继续使用 |

## 6. 约束规则

- `.env` 不提交仓库，只提交 `.example.env`。
- 默认开发目录必须对普通用户可写。
- 文件日志初始化失败时只降级为控制台日志，不能阻断服务启动。
- 生产环境不得依赖开发默认 `.seclab/`。
- 生产环境必须通过 `SECLAB_HOME` 或单项变量明确路径和权限。
- `SECLAB_HOME` 管理配置、数据库、日志和运行态文件，不管理二进制入口。
