# AISpec — AI 工程规范体系

本仓库是一套供 AI 编码助手使用的工程规范:`rules/` 是规则真源,`skills/` 是任务入口。执行编码、审查、脚手架任务时,按下表加载对应规范。

## 域索引

| 域 | 规则 | Skill | 覆盖范围 |
|----|------|-------|---------|
| Go 服务端 | `rules/go-server/` | `go-server` | HTTP API、gRPC、消息消费、定时任务、Worker |
| .NET 服务端 | `rules/dotnet-server/` | `dotnet-server` | ASP.NET Core API、微服务、Worker Service |
| Python 服务端 | `rules/python-server/` | `python-server` | FastAPI / Django / Flask |
| Node.js 服务端 | `rules/node-server/` | `node-server` | NestJS / Express / Fastify |
| Rust | `rules/rust/` | `rust` | axum 服务端、CLI、库(Tauri 后端叠加 tauri-desktop) |
| 前端 | `rules/frontend/` | `frontend` | Vue3 后台管理、uni-app H5 / 微信小程序 |
| Tauri 桌面 | `rules/tauri-desktop/` | `tauri-desktop` | Tauri v2 + Rust 跨平台桌面 |
| .NET 桌面 | `rules/dotnet-desktop/` | `dotnet-desktop` | WPF / MAUI / WinForms |
| Flutter | `rules/flutter/` | `flutter` | Dart + Flutter 跨平台应用 |
| Android | `rules/android/` | `android` | Kotlin + Jetpack Compose |
| iOS | `rules/ios/` | `ios` | Swift + SwiftUI |
| 数据库 | `rules/database.md` | `database` | Schema、迁移脚本、种子数据 |

## 跨域规范

| 规范 | 文件 | 何时加载 |
|------|------|---------|
| 安全基线 | `rules/security.md` | 认证、密钥、敏感数据、依赖安全 |
| 前后端协作 | `rules/collaboration.md` | API 契约、版本管理、错误码、联调、发布 |
| 设计规范 | `rules/design.md` | UI 设计、视觉、交互、无障碍、响应式 |
| 测试规范 | `rules/testing.md` | E2E 测试、性能测试 |
| 运维基线 | `rules/operations.md` | 可观测性、环境管理、发布回滚 |

## 横切 Skills

| Skill | 用途 |
|-------|------|
| `spec-generator` | 新项目技术规格说明书(五阶段引导) |
| `security-auditor` | 安全审计(OWASP、密钥、依赖、威胁建模) |
| `design-guide` | UI/UX 设计方案与设计走查 |
| `rules-maintainer` | 本仓库规范文件的维护与校验 |

## 通用编码约束(所有域必遵,MUST)

1. **函数单一职责**:一个函数只做一件事;函数超过约 50 行、嵌套超过 3 层、或需要用「和/并且」才能描述清楚职责时必须拆分;优先早返回(guard clause)减少嵌套。
2. **文件单一职责**:一文件一责任,文件名体现职责;禁止 `util`/`common`/`misc`/`helper` 类杂物文件承载多责任逻辑。
3. **参数与返回**:参数超过 5 个用对象/结构体聚合;禁止用布尔参数切换函数行为(拆成两个函数)。
4. **重复治理**:同一逻辑第三次出现必须抽象复用(rule of three);禁止复制粘贴后微调的平行实现。
5. **命名表达意图**:名称说清「做什么」,不依赖注释补救;禁止无意义缩写与 `data`/`info`/`temp` 类空洞命名。

## 加载策略

1. 每个域规则分两个文件:**`baseline.md` 必载**(硬约束 + 红线,任何该域任务都先读),`practices.md` 按场景章节选读。
2. 跨域任务按涉及的域分别加载各自 baseline;涉及 API 契约加 `collaboration.md`,涉及数据库结构加 `database.md`,涉及敏感数据加 `security.md`。
3. 单次任务加载规则文件控制在 2~5 个,不要一次性通读全部。
4. 输出代码前对照所在域 baseline 的「红线」清单自查。

## 冲突仲裁(MUST)

优先级从高到低:

1. `rules/database.md` — 数据库结构与迁移条款全局最高
2. `rules/security.md` — 安全要求优先于便利性
3. `rules/collaboration.md` — 跨端契约、联调、发布顺序
4. 其余跨域规范(design / testing / operations)
5. 各域内部规则(域内:baseline 红线 > practices 指引)

同级冲突以「更严格且可验证」的条款为准;仍无法消解时向用户说明并请其裁定。

## 规范变更

修改本仓库规则/skill 时使用 `rules-maintainer` skill:遵守结构契约(域规则两文件、跨域单文件、SKILL.md 标准 frontmatter)、行数预算与引用完整性校验。
