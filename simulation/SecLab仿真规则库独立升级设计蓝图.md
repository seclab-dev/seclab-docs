# SecLab 仿真规则库独立升级设计规范

本文档定义了 `seclab-sim-rules` 运行时规则包升级方案的技术架构与实现规范，用以指导规则库的独立打包发布、主控的安全验签、数据交互接口及底层持久化存储的统一开发。

---

## 1. 架构概述

系统采用运行时动态升级架构。协议仿真规则资产与主控程序完全解耦，主控在运行时通过管理端 API 接口导入规则库包，进行安全验签与内容审计后，动态同步并应用于各计算节点。

### 1.1 模块职责划分

- **seclab-sim-rules (规则库项目)**：负责规则源数据管理、YAML 配置的 Schema 静态审计、Protobuf 格式转换、签名生成以及规则包的自动化发布。
- **seclab (控制面服务)**：提供规则库上传、历史版本查询等管理接口，执行内存签名校验、最低运行版本兼容性核对、ID 范围安全过滤，并通过数据库事务保证规则库状态与实例引用的一致性。
- **seclab-agent (终端服务)**：消费由控制面派发的规则实例指令（包含网络协议与仿真配置），不直接参与规则库包的编解码与验签逻辑。

---

## 2. 规则包格式与编解码规范

### 2.1 物理文件结构

升级规则包为单一的 `.slrp` 文件，含义为 `SecLab Rule Package`。`.slrp` 是协议仿真规则库的专用交付后缀，内部载荷仍然是 gzip 压缩的 tar 归档，解包数据直接读取至内存，包含且仅包含以下两个文件：

```text
seclab-sim-rules-{version}.slrp
  ├── rules.bin        <- 内存 Protobuf 序列化二进制载荷
  └── rules.bin.sig    <- 基于 rules.bin 原始字节的 Ed25519 detached 签名文本
```

规则包的校验和解析全程在主控服务内存中完成，不落盘产生临时文件，确保解析边界的运行时安全。主控导入时必须校验 `.slrp` 后缀、归档可解析性、包内文件集合和签名，不应只信任文件名。

### 2.2 内存反序列化协议 (Protobuf)

`rules.bin` 的数据载荷通过以下 Protobuf 契约进行序列化与反序列化：

```protobuf
syntax = "proto3";

// 规则库元数据
message RulePackageManifestProto {
    string package_id = 1;              // 规则库唯一标识 (例如 "seclab-sim-rules")
    string version = 2;                 // 规则库语义化版本号
    int32 ruleset_format_version = 3;   // 规则集格式版本号
    string min_seclab_version = 4;      // 要求的最低主控版本号
    int64 generated_at = 5;             // 规则库包生成的时间戳 (Unix Epoch)
    int32 rule_count = 6;               // 包内规则总数
}

// 仿真规则具体字段
message SimRuleProto {
    int64 id = 1;                       // 规则唯一标识 ID
    string name = 2;                    // 中文规则名
    string name_en = 3;                 // 英文规则名
    string cve = 4;                     // 关联 CVE 编号
    string category = 5;                // 规则分类
    string description_zh = 6;          // 中文描述
    string description_en = 7;          // 英文描述
    string protocol = 8;                // 网络仿真协议类型 (如 "http")
    int64 default_port = 9;             // 默认仿真监听端口
    string config_json = 10;            // 仿真具体的详细配置 (结构化 JSON 字符串)
}

// 统一包装规则包
message SimRulePackageProto {
    RulePackageManifestProto manifest = 1;
    repeated SimRuleProto rules = 2;
}
```

---

## 3. 安全与信任模型

系统强制启用签名验证机制，以此保障平台规则资产导入的真实性与完整性。

- **内置公钥**：复用主控 `seclab-upgrade::signing_key::SECLAB_RELEASE_PUBLIC_KEY` 编译内置公钥（Minisign 格式）。
- **算法基础**：基于 Minisign 兼容的 Ed25519 签名算法体系。
- **校验生命周期**：在内存中将提取出的 `rules.bin` 原始字节与 `rules.bin.sig` 中的签名文本执行 Ed25519 签名真实性校验，签名验证通过后方可反序列化并读取规则内容。

---

## 4. 导入与校验控制流程

控制面接收到规则包上传请求后，按照以下正向控制流进行校验和状态变更：

1. **解包提取**：在内存中读取 `rules.bin` 与 `rules.bin.sig`。
2. **签名验签**：对 `rules.bin` 进行 Ed25519 签名验证，确保其未被篡改。
3. **数据反序列化**：解码 Protobuf 结构，提取 `manifest` 及规则列表。
4. **包重复性校验 (幂等机制)**：
   - 检查 `sim_rule_packages` 表中是否存在相同的 `(package_id, version)` 记录。
   - 若已存在，则**直接跳过数据库更新操作**（接口响应 `skipped: true`），保持系统状态幂等，避免多余写入与规则覆盖。
5. **合规与兼容性验证**：
   - 使用 `semver` 规范校验当前主控运行版本是否满足 `min_seclab_version`。
   - 验证反序列化后的规则数量是否与 `manifest.rule_count` 声明严格一致。
   - **ID 分段安全保护**：包规则 ID 必须小于 `1,000,000`（ID `>= 1,000,000` 保留供用户在界面创建的自定义规则使用），凡出现越界 ID 立即阻断导入。
6. **数据库事务写入**：
   - 将该 `package_id` 下所有其他旧包的运行状态标记为 `superseded`。
   - 写入当前新包元数据至 `sim_rule_packages` 表，状态设为 `active`。
   - 将包规则逐一 UPSERT 写入 `sim_rules` 表，设置 `source_type = 'package'`，状态设为 `active`。
   - **缺失规则逻辑停用**：提取本地已存在但新规则包中缺失的同包规则 ID 集合，将这些规则在 `sim_rules` 中的 `rule_status` 更新为 `inactive`，防止直接物理删除导致历史仿真实例或交互审计日志关联失效。
7. **数据落盘备份**：
   - 将 `rules.bin` 的原始二进制字节 physical 备份归档至主控本地的 `sim-rules` 配置目录下（命名为 `seclab-sim-rules-{version}.bin`）。
8. **派发事件日志**：
   - 异步登记平台事件日志（`PlatformLogEntry`），事件类型为 `simulation_rule_package_import`，包含是否触发跳过升级（`skipped`）等元数据。

---

## 5. 数据库设计 (SQLite)

### 5.1 `sim_rules` 表

用于管理仿真规则的配置与来源存续状态：

```sql
CREATE TABLE IF NOT EXISTS sim_rules (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    name_en TEXT NOT NULL,
    cve TEXT,
    category TEXT NOT NULL,
    description_zh TEXT NOT NULL,
    description_en TEXT NOT NULL,
    protocol TEXT NOT NULL,
    default_port INTEGER,
    config_yaml TEXT NOT NULL,
    source_type TEXT NOT NULL DEFAULT 'custom',   -- 'package' 或 'custom'
    source_package_id TEXT,                      -- 关联的规则包唯一 ID
    rule_status TEXT NOT NULL DEFAULT 'active',   -- 'active' (已启用) 或 'inactive' (已停用)
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);
CREATE INDEX IF NOT EXISTS idx_sim_rules_status_type ON sim_rules (rule_status, source_type);
```

### 5.2 `sim_rule_packages` 表

用于记录导入的历史规则库元数据：

```sql
CREATE TABLE IF NOT EXISTS sim_rule_packages (
    package_id TEXT NOT NULL,
    version TEXT NOT NULL,
    ruleset_format_version INTEGER NOT NULL,
    min_seclab_version TEXT NOT NULL,
    rule_count INTEGER NOT NULL,
    signature_hex TEXT NOT NULL,
    archive_sha256 TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('active', 'superseded')), -- 'active' (当前激活), 'superseded' (已被更替)
    imported_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    PRIMARY KEY (package_id, version)
);
```

---

## 6. API 接口规范

### 6.1 导入规则库

- **路径**：`POST /api/v1/simulation/rule-package/import`
- **Content-Type**：`multipart/form-data`
- **请求参数**：`archive` (外层 `.slrp` 文件)
- **响应载荷**：
  - **全新规则库版本升级成功** (HTTP 200，消息键 `app.simulation.rules.messages.packageImportSuccess`)：

    ```json
    {
      "success": true,
      "code": 200,
      "message": "Rules package imported successfully",
      "data": {
        "packageId": "seclab-sim-rules",
        "version": "0.1.0-alpha.1",
        "rulesetFormatVersion": 1,
        "minSeclabVersion": "0.1.0-alpha.1",
        "ruleCount": 30,
        "signatureHex": "RUQbTVkt...",
        "archiveSha256": "4f71d252...",
        "status": "active",
        "importedAt": "2026-06-18T00:54:46.645661124+00:00"
      }
    }
    ```

  - **检测到版本一致直接跳过** (HTTP 200，消息键 `app.simulation.rules.messages.packageImportAlreadyLatest`)：

    ```json
    {
      "success": true,
      "code": 200,
      "message": "Rules package is already up to date",
      "data": {
        "packageId": "seclab-sim-rules",
        "version": "0.1.0-alpha.1",
        ...
      }
    }
    ```

### 6.2 查询历史已导入规则库包

- **路径**：`GET /api/v1/simulation/rule-packages/list`
- **响应载荷**：

  ```json
  {
    "success": true,
    "code": 200,
    "message": "Rule packages historical records loaded",
    "data": [
      {
        "packageId": "seclab-sim-rules",
        "version": "0.1.0-alpha.1",
        ...
      }
    ]
  }
  ```

### 6.3 查询当前激活规则库

- **路径**：`GET /api/v1/simulation/rule-package/current`
- **响应载荷**：返回当前数据库中状态为 `active` 且 `package_id = 'seclab-sim-rules'` 的规则库信息。

---

## 7. 前端页面交互与日志对接规范

### 7.1 事件日志对接

- 规则包的导入流程统一产生平台事件日志，事件标识为 `simulation_rule_package_import`。
- 在“事件日志”列表中该事件类型翻译呈现为 **“导入规则库”**。
- 日志扩展详情中以结构化 JSON 数据展示：包版本、规则库 ID、规则数量以及是否跳过（`skipped`）。
