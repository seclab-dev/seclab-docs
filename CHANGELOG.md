# Changelog

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，并遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [Unreleased]

### Changed

- 补充套件平台运行契约版本设计，覆盖清单声明、导入校验、实例固化和 SecLab 升级兼容性检查。
- 重写协议仿真 v1 设计文档，覆盖 14 种协议、具名 TCP/UDP 多端点、DNS 双传输和 engine 运行时事件。
- 将规则库蓝图收敛为当前规则包规范，补齐 Protobuf v1、真实签名校验、`dist/` 交付目录和全链路兼容性契约。
- 同步套件系统、Compose 开发与 Runtime SDK 文档，明确平台最低版本校验、服务/能力授权、workload 镜像白名单和整 workload 抓包。
- 在 Agent OpenAPI 中登记 Suite Runtime workload 与 capture 接口及其实际请求、响应和安全边界。

## [0.1.0-alpha.1] - 2026-06-28

### Added

- 首次发布 SecLab Dev 组织级文档仓库。
- 提供架构、运行规范、API 治理、套件规范、协议仿真和在线升级文档。
- 提供套件 SDK 设计规范，统一主控与套件 Web 前端之间的通信协议。
- 提供 `linked-docs` 接入约定，支持业务仓库通过本地软链接共享统一文档源。
- 提供 API 规范、契约和治理文档，用于支撑接口目录和可视化工作台。
