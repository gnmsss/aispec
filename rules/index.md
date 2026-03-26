# rules/index.md

## 目的
1. 作为整个工程规范体系的唯一顶层入口，索引所有规范域及其关系。
2. 定义跨域冲突仲裁规则，消除各规范域之间优先级模糊地带。

## 规范域概览

| 域 | 入口文件 | 覆盖范围 | 编码引导 Skill | 规则维护 Skill |
|----|---------|---------|---------------|---------------|
| Go 服务端 | `go-server/index.md` | HTTP API、gRPC、消息消费、定时任务、Worker | `$go-server-coding-guide` | `$go-server-rules-maintainer` |
| .NET 服务端 | `dotnet-server/index.md` | ASP.NET Core API、微服务、单体应用 | `$dotnet-server-coding-guide` | `$dotnet-server-rules-maintainer` |
| 前端 | `frontend/index.md` | 后台管理、公众号 H5、小程序 | `$frontend-coding-guide` | `$frontend-rules-maintainer` |
| .NET 桌面 | `dotnet-desktop/index.md` | WPF、MAUI、WinForms 桌面应用 | `$dotnet-desktop-coding-guide` | `$dotnet-desktop-rules-maintainer` |
| Tauri 桌面 | `tauri-desktop/index.md` | Rust + Tauri 跨平台桌面应用 | `$tauri-desktop-coding-guide` | `$tauri-desktop-rules-maintainer` |
| Android 移动端 | `android/index.md` | Kotlin/Java Android 原生应用 | `$android-coding-guide` | `$android-rules-maintainer` |
| iOS 移动端 | `ios/index.md` | Swift/ObjC iOS 原生应用 | `$ios-coding-guide` | `$ios-rules-maintainer` |
| Flutter 跨平台 | `flutter/index.md` | Dart + Flutter 跨平台应用（移动/桌面/Web） | `$flutter-coding-guide` | `$flutter-rules-maintainer` |
| Python 服务端 | `python-server/index.md` | FastAPI / Django / Flask 服务端 | `$python-server-coding-guide` | `$python-server-rules-maintainer` |
| Java 服务端 | `java-server/index.md` | Spring Boot / Spring Cloud 服务端 | `$java-server-coding-guide` | `$java-server-rules-maintainer` |
| Node.js 服务端 | `node-server/index.md` | NestJS / Express / Fastify 服务端 | `$node-server-coding-guide` | `$node-server-rules-maintainer` |
| Electron 桌面 | `electron-desktop/index.md` | Electron + React/Vue 桌面应用 | `$electron-desktop-coding-guide` | `$electron-desktop-rules-maintainer` |
| React Native | `react-native/index.md` | React Native 跨平台移动应用 | `$react-native-coding-guide` | `$react-native-rules-maintainer` |
| 前后端协作 | `frontend-backend-collaboration.md` | API 契约、联调、发布回滚 | `$frontend-backend-coding-guide` | `$frontend-backend-rules-maintainer` |
| 数据库 | `database/index.md` | Schema 初始化、迁移脚本、种子数据 | `$database-coding-guide` | `$database-rules-maintainer` |
| Monorepo | `monorepo/monorepo.md` | Monorepo 工程管理、构建缓存、CI 优化 | — | — |
| 安全基线 | `security/security-baseline.md` | 认证授权、数据安全、输入校验、密钥管理 | — | — |
| 环境管理 | `environment/environment-management.md` | 多环境配置、密钥管理、功能开关 | — | — |
| 可观测性 | `observability/observability.md` | 指标、日志、链路追踪、告警、SLO | — | — |
| API 版本管理 | `api-versioning/api-versioning.md` | API 版本策略、兼容性、废弃流程 | — | — |
| 版本发布 | `release/release-management.md` | SemVer、Changelog、Release Notes、发布流程 | — | — |
| E2E 测试 | `testing/e2e-testing.md` | E2E 测试策略、POM、CI 集成 | — | — |
| 性能测试 | `testing/performance-testing.md` | 测试类型、性能指标、基线、场景设计、CI 集成 | — | — |
| 国际化 | `i18n/internationalization.md` | 文案管理、日期时间、货币、RTL | — | — |
| 设计规范 | `design/index.md` | 视觉美学、设计系统、交互原则、无障碍、响应式、错误码体系 | — | — |

## 加载策略

本体系支持两种使用模式，共享同一套规则文件：

### 模式 A：单体模式（单个 AI 实例直接调用 skill）

#### 编码任务加载策略（AI 编写代码 / 指导开发者）
1. 跨域任务使用 `$task-router` 分析涉及的域并确定执行顺序。
2. 每个域使用对应的 `*-coding-guide` 按编码场景加载 2~5 个规则文件。
3. 始终加载底线规则（每个域的 `baseline.md` + `forbidden.md`），按场景追加命中文件。
4. 跨域交互时由 `$frontend-backend-coding-guide` 确保契约一致性。
5. 每个编码场景加载的规则文件总量控制在 3~6 个，不超过 8 个。

#### 规则维护加载策略（维护规则文件本身）
1. 按需加载：每次任务仅读取命中的规范文件，禁止一次性通读全部。
2. 入口优先：先读对应域的 `index.md`，再按索引读取细分文件。
3. 跨域任务：涉及多域时，先读 `frontend-backend-collaboration.md`，再按需读取各域细分文件。

### 模式 B：多 Agent 模式（多个专业化 Agent 协作）
1. **入口**：`agents/index.md` — Agent 模式总入口与索引。
2. Coordinator Agent 接收用户任务，使用 `$task-router` 识别涉及的域。
3. 当 Spec 和 Design 产出可用时，Coordinator 使用 `$task-planner` 将 Spec + Design 产出拆解为按域分组的可执行任务清单。
4. 每个域由独立的域 Agent 执行，域 Agent 内部调用对应的 skill 加载规则。
5. Agent 之间通过标准化协议协作：`agents/protocols/coordination.md`（调度）、`agents/protocols/handoff.md`（交接）、`agents/protocols/agent-output-format.md`（输出格式）、`agents/protocols/execution-trace.md`（执行追溯）。
6. 跨域冲突仲裁规则与单体模式一致（见下方"跨域冲突仲裁"章节）。
7. 各平台适配指南见 `agents/adapters/` 及项目根目录 `README.md`。

### 执行追溯输出要求（MUST）
无论使用哪种模式，每次任务执行完毕后必须在输出末尾附**执行追溯摘要**，告知用户本次调用了哪个 Agent / Skill、加载了哪些规则文件、执行顺序和跨域交接情况。格式定义见 `agents/protocols/execution-trace.md`。

## 跨域冲突仲裁（MUST）

### 层级优先级（从高到低）
1. `rules/database/database.md` — 数据库结构与迁移相关条款拥有最高优先级。
2. `rules/security/security-baseline.md` — 安全基线条款，安全要求优先于便利性。
3. `rules/frontend-backend-collaboration.md` — 跨端接口契约、联调、发布顺序相关条款。
4. 跨域规范 — `observability/`、`environment/`、`api-versioning/`、`release/`、`i18n/`、`testing/` 中的条款。
5. 各域内部规则 — 各技术栈域内条款（Go / .NET / Python / Java / Node.js 服务端、前端、.NET / Tauri / Electron 桌面、Android、iOS、Flutter、React Native）。

### 域内优先级
1. Go 服务端：`profiles/* > common/*`。
2. 前端：`applications/* > common/*`；端侧结构文件 > 通用结构文件。
3. .NET 服务端：`profiles/* > common/*`。
4. Python 服务端：`profiles/* > common/*`。
5. Java 服务端：`profiles/* > common/*`。
6. Node.js 服务端：`profiles/* > common/*`。
7. .NET 桌面：`profiles/* > common/*`。
8. Tauri 桌面：`profiles/* > common/*`。
9. Android 移动端：`profiles/* > common/*`。
10. iOS 移动端：`profiles/* > common/*`。
11. Flutter 跨平台：`profiles/* > common/*`。
12. Electron 桌面：`profiles/* > common/*`。
13. React Native：`profiles/* > common/*`。

### 冲突场景仲裁表

| 冲突场景 | 仲裁规则 | 示例 |
|---------|---------|------|
| 数据库迁移 vs 任意域 | 以 `database/database.md` 为准 | 迁移脚本格式、严禁修改历史脚本 |
| API 响应结构：Go 端 vs 协作规范 | 以 `frontend-backend-collaboration.md` 为准 | 错误码语义、响应字段命名 |
| 发布顺序：Go 端 vs 前端 | 以 `frontend-backend-collaboration.md` 为准 | 先服务端兼容发布，再前端切换 |
| Go profile vs Go common | 以 profile 为准 | 微服务禁止共享 ORM model |
| 前端应用端 vs 前端通用 | 以应用端为准 | 小程序禁 SVG、2MB 主包上限 |
| 同级规则冲突 | 以"更严格且可验证"的条款为准 | — |
| 仍无法消解 | 提交评审记录，由技术负责人裁定 | — |

### 仲裁原则
1. 越靠近数据真源的规则优先级越高。
2. 跨端约束优先于单端实现细节。
3. 可验证的硬约束优先于描述性建议。
4. 任何例外必须在评审中记录原因、边界、回收时间，并绑定责任人。

## 规范版本策略
1. 每次规范变更必须通过对应域的校验脚本（结构校验 + 语义校验）。
2. 跨域变更必须同时通过涉及域的所有校验脚本。
3. 非兼容规范变更须附迁移指引，说明已有项目的升级步骤。

---
<!-- AI: 以下为目录结构参考，执行任务时无需加载，可在此停止读取 -->

## 目录结构
```
rules/
├── index.md                              ← 本文件（顶层入口）
├── go-server.md                      ← Go 服务端兼容入口
├── go-server/
│   ├── index.md                          ← Go 服务端规则总入口
│   ├── common/                           ← 16 个通用规则文件
│   └── profiles/                         ← 单体 + 微服务 profile
├── frontend.md                           ← 前端兼容入口
├── frontend/
│   ├── index.md                          ← 前端规则总入口
│   ├── quickstart.md                     ← 新人快速上手指南
│   ├── common/                           ← 15 个通用规则文件
│   ├── applications/                     ← 3 个应用端规则
│   ├── project-structure/                ← 3 个项目结构规则
│   ├── frameworks/                       ← 框架参考规则
│   └── matrix/                           ← 历史兼容
├── frontend-backend-collaboration.md     ← 前后端协作规范
├── dotnet-server.md                      ← .NET 服务端兼容入口
├── dotnet-server/
│   ├── index.md                          ← .NET 服务端规则总入口
│   ├── common/                           ← 16 个通用规则文件
│   └── profiles/                         ← 单体 + 微服务 profile（含 8 个微服务专题）
├── dotnet-desktop.md                     ← .NET 桌面兼容入口
├── dotnet-desktop/
│   ├── index.md                          ← .NET 桌面规则总入口
│   ├── common/                           ← 13 个通用规则文件
│   └── profiles/                         ← WPF + MAUI + WinForms profile
├── tauri-desktop.md                      ← Tauri 桌面兼容入口
├── tauri-desktop/
│   ├── index.md                          ← Tauri 桌面规则总入口
│   ├── common/                           ← 12 个通用规则文件
│   └── profiles/                         ← Tauri v2 profile
├── android.md                            ← Android 移动端兼容入口
├── android/
│   ├── index.md                          ← Android 规则总入口
│   ├── common/                           ← 12 个通用规则文件
│   └── profiles/                         ← Compose + XML Views profile
├── ios.md                                ← iOS 移动端兼容入口
├── ios/
│   ├── index.md                          ← iOS 规则总入口
│   ├── common/                           ← 12 个通用规则文件
│   └── profiles/                         ← SwiftUI + UIKit profile
├── flutter.md                            ← Flutter 跨平台兼容入口
├── flutter/
│   ├── index.md                          ← Flutter 规则总入口
│   ├── common/                           ← 13 个通用规则文件
│   └── profiles/                         ← Mobile profile
├── python-server.md                      ← Python 服务端兼容入口
├── python-server/
│   ├── index.md                          ← Python 服务端规则总入口
│   ├── common/                           ← 16 个通用规则文件
│   └── profiles/                         ← 单体 + 微服务 profile
├── java-server.md                        ← Java 服务端兼容入口
├── java-server/
│   ├── index.md                          ← Java 服务端规则总入口
│   ├── common/                           ← 16 个通用规则文件
│   └── profiles/                         ← 单体 + 微服务 profile
├── node-server.md                        ← Node.js 服务端兼容入口
├── node-server/
│   ├── index.md                          ← Node.js 服务端规则总入口
│   ├── common/                           ← 16 个通用规则文件
│   └── profiles/                         ← 单体 + 微服务 profile
├── electron-desktop.md                   ← Electron 桌面兼容入口
├── electron-desktop/
│   ├── index.md                          ← Electron 桌面规则总入口
│   ├── common/                           ← 12 个通用规则文件
│   └── profiles/                         ← Electron v30 profile
├── react-native.md                       ← React Native 兼容入口
├── react-native/
│   ├── index.md                          ← React Native 规则总入口
│   ├── common/                           ← 13 个通用规则文件
│   └── profiles/                         ← Expo + Bare profile
├── design/
│   ├── index.md                          ← 设计规范总入口
│   ├── aesthetics.md                     ← 美学指南
│   ├── design-system.md                  ← 设计系统规范
│   ├── ux-principles.md                  ← 交互设计原则
│   ├── accessibility.md                  ← 无障碍设计规范
│   ├── responsive.md                     ← 响应式设计规范
│   └── error-code-system.md              ← 错误码体系规范
├── database/
│   ├── index.md                          ← 数据库规则总入口
│   ├── database.md                       ← 数据库 Schema 与迁移
│   └── data-migration.md                 ← 数据迁移与种子数据规范
├── monorepo/
│   └── monorepo.md                       ← Monorepo 工程规范
├── security/
│   └── security-baseline.md              ← 安全基线规范
├── environment/
│   └── environment-management.md         ← 环境管理与配置规范
├── observability/
│   └── observability.md                  ← 监控与可观测性规范
├── api-versioning/
│   └── api-versioning.md                 ← API 版本管理规范
├── release/
│   └── release-management.md             ← 版本发布管理规范
├── testing/
│   ├── e2e-testing.md                    ← E2E 测试规范
│   └── performance-testing.md            ← 性能测试规范
├── i18n/
│   └── internationalization.md           ← 国际化规范
└── templates/                            ← 可复用规范模板（参见 templates/index.md）
    ├── index.md                          ← 模板总索引（谁用/何时用/怎么用）
    ├── exception-request-template.md     ← 规范例外申请模板（通用）
    ├── go-server/                    ← Go 服务端专用模板
    │   └── pr-review-checklist.md        ← Go 服务端 PR 评审清单
    ├── dotnet-server/                    ← .NET 服务端专用模板
    │   └── pr-review-checklist.md        ← .NET 服务端 PR 评审清单
    ├── dotnet-desktop/                   ← .NET 桌面专用模板
    │   └── pr-review-checklist.md        ← .NET 桌面 PR 评审清单
    ├── tauri-desktop/                    ← Tauri 桌面专用模板
    │   └── pr-review-checklist.md        ← Tauri 桌面 PR 评审清单
    ├── android/                          ← Android 移动端专用模板
    │   └── pr-review-checklist.md        ← Android PR 评审清单
    ├── ios/                              ← iOS 移动端专用模板
    │   └── pr-review-checklist.md        ← iOS PR 评审清单
    ├── python-server/                    ← Python 服务端专用模板
    │   └── pr-review-checklist.md        ← Python 服务端 PR 评审清单
    ├── java-server/                      ← Java 服务端专用模板
    │   └── pr-review-checklist.md        ← Java 服务端 PR 评审清单
    ├── node-server/                      ← Node.js 服务端专用模板
    │   └── pr-review-checklist.md        ← Node.js 服务端 PR 评审清单
    ├── electron-desktop/                 ← Electron 桌面专用模板
    │   └── pr-review-checklist.md        ← Electron 桌面 PR 评审清单
    ├── react-native/                     ← React Native 专用模板
    │   └── pr-review-checklist.md        ← React Native PR 评审清单
    ├── flutter/                          ← Flutter 跨平台专用模板
    │   └── pr-review-checklist.md        ← Flutter PR 评审清单
    ├── frontend/                         ← 前端专用模板
    │   ├── pr-review-checklist.md        ← 前端 PR 评审清单
    │   ├── eslint-prettier-baseline.md   ← ESLint / Prettier 配置基线
    │   ├── permission-naming.md          ← 权限点命名规范（admin-console）
    │   ├── uni-request-wrapper.md        ← uni.request 标准封装（H5 + 小程序）
    │   ├── miniprogram-review-checklist.md ← 小程序审核清单
    │   ├── tailwind-element-plus.md      ← Tailwind + Element Plus 组合约束
    │   ├── wechat-auth-share-flow.md     ← 微信授权与分享流程规范
    │   ├── pro-table.md                  ← Schema-Driven 表格（ProTable）
    │   ├── tiptap-editor.md              ← Tiptap 富文本编辑器封装
    │   ├── dependency-management.md      ← 依赖管理与脚手架依赖清单
    │   ├── component-patterns.md         ← 三端组件示例与适配层
    │   ├── normalization-toolkit.md      ← 规范化改造工具包
    │   ├── workflow-toolkit.md           ← 交付流程工具包（PR 模板 + 页面行数检查）
    │   ├── miniprogram-ci-checks.md      ← 小程序 CI 检查（包体积 + 资源格式）
    │   ├── naming-toolkit.md             ← 命名规范工具包（文件命名 + Token 冲突）
    │   ├── wechat-h5-toolkit.md          ← 微信 H5 工具包（兼容测试 + 活动归档）
    │   ├── git-workflow-config.md        ← commitlint / husky / 分支保护配置
    │   ├── testing-toolkit.md            ← Vitest 配置 / testing-library 样板
    │   ├── security-toolkit.md           ← CSP / DOMPurify / 依赖审计脚本
    │   └── ci-pipeline.md               ← GitHub Actions 完整 CI 流水线
    ├── database/                         ← 数据库专用模板
    │   └── pr-review-checklist.md        ← 数据库 PR 评审清单
    └── frontend-backend/                 ← 前后端协作模板
        ├── api-contract-template.md      ← API 接口契约模板
        ├── integration-checklist-template.md ← 联调检查清单
        └── release-rollback-record-template.md ← 发布回滚记录
```
