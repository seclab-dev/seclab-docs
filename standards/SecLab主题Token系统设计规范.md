# SecLab 主题 Token 系统设计规范

本文档定义 SecLab Design Language（SDL）的主题 Token 边界和使用规则。Token 的唯一源码位于 `seclab-ui` 仓库的 `packages/tokens/`，通过 `@seclab-dev/tokens` 发布。消费项目不得复制或维护另一份公共 Token。

## 1. 设计原则

- 工程化：结构清晰、状态明确、操作可预测。
- 控制面气质：适合节点、任务、资源、日志和安全实验管理。
- 信息密度：优先可扫描性，不使用营销式页面构图。
- 浅色默认：主控默认使用浅色主题，同时完整支持深色主题。
- 克制表达：禁止卡通黑客元素、霓虹字符雨、装饰性光球和大面积高饱和渐变。

## 2. 事实来源与职责

| 内容 | 唯一来源 |
| --- | --- |
| 公共主题变量 | `@seclab-dev/tokens` |
| Vue 基础组件样式 | `@seclab-dev/vue/style.css` |
| 自有 SVG 图标 | `@seclab-dev/icons` |
| 主控专属资源变量 | 主控 `frontend/src/styles/theme.css` |

公共 Token 不得引用消费项目的图片、字体或其他静态资源路径。桌面壁纸等主控专属变量由主控自行定义。

## 3. 接入方式

```css
@import '@seclab-dev/tokens/index.css';
@import '@seclab-dev/vue/style.css';
```

浅色主题通过根元素属性启用：

```html
<html data-theme="light">
```

深色主题使用 `data-theme="dark"` 或移除浅色覆盖。主控运行时默认写入 `data-theme="light"`。

## 4. Token 分类

- 背景：`--sdl-bg-*`
- 文本：`--sdl-text-*`
- 品牌：`--sdl-primary`、`--sdl-secondary`、`--sdl-accent`
- 状态：`--sdl-success`、`--sdl-warning`、`--sdl-danger`、`--sdl-info`
- 边框：`--sdl-border-*`
- 字体：`--sdl-font-*`
- 间距：`--sdl-space-*`
- 圆角：`--sdl-radius-*`
- 阴影：`--sdl-shadow-*`
- 层级：`--sdl-z-index-*`

具体变量和值以已发布包及 `seclab-ui/packages/tokens/index.css` 为准，本文档不重复维护完整数值表。

## 5. 使用规则

1. 业务组件优先使用现有 Token，不散落硬编码主题颜色。
2. 成功或在线使用 `success`，警告或排队使用 `warning`，失败或危险操作使用 `danger`，提示和通信状态使用 `info` 或 `primary`。
3. 日志、路径、命令、哈希、端口和 IP 使用等宽字体 Token。
4. 工具栏和紧凑表单通常使用 `--sdl-space-2` 或 `--sdl-space-3`；主内容区域通常使用 `--sdl-space-4`。
5. 消费项目需要新增公共变量时，应先在 `seclab-ui` 中设计、发布，再升级依赖；不得只在单个项目中创建同名公共 Token。
6. 项目专属变量应保持明确边界，例如主控的 `--sdl-desktop-wallpaper`。

## 6. 组件约束

- 业务 UI 优先使用 `@seclab-dev/vue` 提供的 `SecLab*` 组件。
- 不引入 Element Plus、Ant Design、Naive UI 等并行组件体系。
- 图标优先使用 `SecLabIcon` 和 `@seclab-dev/icons`。
- 组件 Props、事件和类型声明以发布包 TypeScript 类型为准。

## 7. 发布与升级

四个 UI 包独立维护版本。Token 发生变化时：

1. 更新并发布 `@seclab-dev/tokens`。
2. 如 `@seclab-dev/vue` 需要消费新版本，同步升级其精确依赖并发布 Vue 包。
3. 消费项目使用精确版本升级依赖和锁文件。
4. 执行项目格式化、类型检查、Lint 和生产构建。
