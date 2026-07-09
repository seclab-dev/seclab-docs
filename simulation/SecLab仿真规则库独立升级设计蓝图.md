# SecLab 协议仿真规则包设计规范

本文档定义协议仿真规则包 `.slrp` 的交付格式、导入流程和版本边界。协议仿真已拆分为 `seclab.protocol-simulation` Compose 套件，规则包由套件 API 导入并写入套件私有数据库，主控不再直接解析或持久化协议仿真规则。

## 1. 职责边界

| 仓库或组件 | 职责 |
| --- | --- |
| `seclab-sim-rules` | 维护规则源数据、生成 Protobuf 规则包和签名文件。 |
| `seclab-suite-protocol-simulation` | 解析 `.slrp`，校验规则内容，写入套件私有 SQLite。 |
| `seclab-suites` | 保存协议仿真套件交付清单和固定镜像引用。 |
| `seclab` 主控 | 代理套件 Web 入口，不解析规则包内容。 |

规则包是协议仿真套件的业务资产，不属于主控数据库资产。

## 2. 规则包格式

规则包使用 `.slrp` 后缀，内部是 gzip 压缩 tar 归档。当前套件要求包内包含：

```text
rules.bin
rules.bin.sig
```

导入时必须校验：

1. 上传字段为 `archive` 或 `file`。
2. 归档可解析。
3. `rules.bin` 存在。
4. `rules.bin.sig` 存在。
5. `rules.bin` 可按 Protobuf 解码。
6. manifest 声明的 `rule_count` 与实际规则数量一致。
7. 规则协议在套件支持列表内。
8. 每条规则的 `config_json` 是合法 JSON。

`rules.bin.sig` 当前作为包结构必需文件保留。签名强校验策略可在后续版本加入，但不得改变包内文件命名。

## 3. Protobuf 契约

`rules.bin` 使用以下结构：

```protobuf
syntax = "proto3";

message RulePackageManifestProto {
    string package_id = 1;
    string version = 2;
    int32 ruleset_format_version = 3;
    string min_seclab_version = 4;
    int64 generated_at = 5;
    int32 rule_count = 6;
}

message SimRuleProto {
    int64 id = 1;
    string name = 2;
    string name_en = 3;
    optional string cve = 4;
    string category = 5;
    string description_zh = 6;
    string description_en = 7;
    string protocol = 8;
    optional int64 default_port = 9;
    string config_json = 10;
}

message SimRulePackageProto {
    RulePackageManifestProto manifest = 1;
    repeated SimRuleProto rules = 2;
}
```

字段 tag 不得复用或修改。新增字段只能追加新 tag。

## 4. 套件导入 API

规则包导入由协议仿真套件 API 提供：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/api/rule-package/import` | 导入 `.slrp`。 |
| `GET` | `/api/rule-package/current` | 查询当前规则包。 |

导入成功后返回当前规则包摘要：

```json
{
  "success": true,
  "data": {
    "packageId": "seclab-sim-rules",
    "version": "0.1.0-alpha.1",
    "rulesetFormatVersion": 1,
    "minSeclabVersion": "0.1.0-alpha.1",
    "ruleCount": 30,
    "generatedAt": "2026-07-09T00:00:00Z",
    "importedAt": "2026-07-09T00:00:00Z"
  }
}
```

## 5. 数据落库

套件私有 SQLite 使用：

| 表 | 说明 |
| --- | --- |
| `rule_packages` | 记录当前导入规则包。 |
| `rules` | 保存规则。 |

规则导入时执行事务：

1. 删除旧的包规则。
2. 将每条规则写入 `rules`。
3. 写入或更新 `rule_packages`。

规则 ID 转换为字符串形态：

```text
sim-rule-<rule.id>
```

规则 `config_json` 会被包装为前端和 engine 可消费的 JSON：

```json
{
  "nameEn": "English Name",
  "category": "honeypot",
  "cve": "CVE-2024-0001",
  "description": "中文描述",
  "descriptionEn": "English description",
  "behavior": {}
}
```

## 6. 版本边界

规则包版本、套件版本和 engine 镜像版本是独立版本：

| 版本 | 来源 |
| --- | --- |
| 规则包版本 | `.slrp` manifest 的 `version`。 |
| 套件版本 | `suite.yaml.metadata.version`。 |
| API/UI 镜像版本 | `crates/protocol-simulation/Cargo.toml`。 |
| engine 镜像版本 | `crates/protocol-simulation-engine/Cargo.toml`。 |

规则包使用新协议、新字段或新行为时，必须确认目标协议仿真套件版本已支持对应能力，并更新 `min_seclab_version` 或后续更明确的 `min_suite_version` 字段。
