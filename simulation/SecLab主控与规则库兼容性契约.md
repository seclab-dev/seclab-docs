# 协议仿真套件与规则库兼容性契约

本文档定义 `seclab-suite-protocol-simulation` 与 `seclab-sim-rules` 之间的兼容性约束。协议仿真规则包由套件 API 导入，主控不再直接解析规则包。

## 1. Protobuf Schema 演进

规则库生成端与套件解析端必须保持 Protobuf tag、类型和字段语义一致。

| 端 | 位置 | 结构 |
| --- | --- | --- |
| 套件 | `crates/protocol-simulation/src/rule_package.rs` | `RulePackageManifestProto`、`SimRuleProto`、`SimRulePackageProto` |
| 规则库 | `seclab-sim-rules` | 同名结构 |

允许追加新字段；禁止删除字段、复用 tag、修改既有字段类型或改变既有字段语义。

## 2. `ruleset_format_version`

`ruleset_format_version` 标识规则包格式版本。发生以下变化时必须递增：

- Protobuf 结构出现不兼容变化。
- `.slrp` 内部文件结构变化。
- `config_json` 的业务 schema 出现不兼容变化。

套件导入时必须拒绝不支持的格式版本。

## 3. 版本声明

规则包 manifest 当前包含 `min_seclab_version`。在协议仿真套件化后，该字段表示规则包要求的最低 SecLab 平台能力；当规则依赖套件新能力时，应在规则库发布说明中明确最低协议仿真套件版本。

后续规则包 schema 可以增加 `min_suite_version`，用于精确声明最低协议仿真套件版本。

## 4. 规则 ID

规则库包内 `id` 必须稳定。套件导入时将数字 ID 转为字符串 ID：

```text
sim-rule-<id>
```

规则 ID 一经发布不得复用给不同语义的规则。删除规则时，应通过新规则包移除该规则；套件导入当前版本时会以新包内容覆盖旧包规则。

## 5. 发版顺序

涉及规则包 schema 或协议能力变更时，发布顺序为：

1. 先发布支持新能力的协议仿真套件镜像和 `.slsp` 包。
2. 再发布使用新能力的规则包。
3. 在规则库 CHANGELOG 中记录依赖的最低套件版本。

禁止规则库先于套件使用未支持的新字段、新协议或新配置语义。
