# agents/index.md — Agent 模式总入口

## 目的
1. 定义多 Agent 协作模式的标准化体系，作为 Agent 模式的唯一顶层入口。
2. 索引所有 Agent 定义及其协作协议。
3. 说明 Agent 模式与单体模式的关系。

## 两套模式并存

本规范体系支持两种使用模式，共享同一套 `rules/` 和 `skills/`：

### 模式 A：单体模式（现有）
- 单个 AI 实例通过 `$skill-name` 直接调用 skill。
- Skill 自行按场景加载规则并执行任务。
- 跨域任务通过 `$task-router` 路由。
- 适用场景：Cursor、Codex、Claude Code 等工具中单个 AI 直接响应用户指令。

### 模式 B：多 Agent 模式（本目录定义）
- 每个技术域有一个独立的 Agent，拥有明确的职责边界和可用 skill 集合。
- Coordinator Agent 接收用户任务，识别域并调度域 Agent。
- 域 Agent 之间通过标准化协议（交接、输出契约）协作。
- 适用场景：OpenAI Codex 多 agent、Claude Code agent teams、Gemini ADK 多 agent 编排等。

### 模式选择指南

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 从0到1做一个新产品 | 多 Agent 模式 | 全生命周期：Product→Spec∥Design→开发→QA→DevOps |
| 新项目立项/技术方案设计 | 多 Agent 模式 | Spec Agent 引导完整规格定义 |
| 产品需求/竞品分析 | 多 Agent 模式 | Product Agent 引导竞品调研和 PRD 编写 |
| UI/UX 设计 | 多 Agent 模式 | Design Agent 引导交互和视觉设计 |
| 单域简单任务 | 单体模式 | 直接调用 skill 更高效 |
| 单域复杂任务 | 单体模式 | 一个 skill 即可处理 |
| 跨域业务任务 | 多 Agent 模式 | 需要多域协调和顺序控制 |
| PR 全量审查 | 多 Agent 模式 | 并行审查多域代码变更 |
| 全栈新功能开发 | 多 Agent 模式 | Product→Spec∥Design→DB→服务端→前端按序执行 |
| 测试与质量保障 | 多 Agent 模式 | QA Agent 制定测试策略和执行验收 |
| 安全审计 | 多 Agent 模式 | Security Agent 执行安全威胁建模和合规检查 |
| 部署上线 | 多 Agent 模式 | DevOps Agent 设计 CI/CD 和部署方案 |
| 规则维护/审计 | 按需选择 | 单域维护用单体，跨域审计用多 Agent |

## Agent 索引

### 协调层

| Agent | 定义文件 | 职责 | 可用 Skills |
|-------|---------|------|------------|
| Coordinator | `coordinator/agent.md` | 任务接收、域识别、调度、仲裁、汇总 | `$task-router`、`$task-planner` |

### 产品与设计层

| Agent | 定义文件 | 职责 | 可用 Skills |
|-------|---------|------|------------|
| Product | `product/agent.md` | 竞品分析、需求定义、PRD 编写、路线图 | `$product-prd-writer` |
| Spec | `spec/agent.md` | 项目规格定义、技术选型引导、模块 Spec 生成 | `$spec-generator` |
| Design | `design/agent.md` | UI/UX 设计、交互流程、视觉原型、设计系统 | `$ui-ux-designer`、`$design-reviewer` |

### 质量与运维层

| Agent | 定义文件 | 职责 | 可用 Skills |
|-------|---------|------|------------|
| Security | `security/agent.md` | 安全威胁建模、安全审计、漏洞扫描、合规检查 | `$security-auditor` |
| QA | `qa/agent.md` | 测试策略、测试用例、验收测试、质量报告 | `$qa-test-strategist` |
| DevOps | `devops/agent.md` | CI/CD、部署策略、监控告警、环境管理 | `$devops-engineer` |

### 域 Agent

| Agent | 定义文件 | 覆盖范围 | 可用 Skills |
|-------|---------|---------|------------|
| GoServer | `go-server/agent.md` | Go 服务端（HTTP API、gRPC、Worker） | `$go-server-coding-guide`、`$go-server-code-reviewer`、`$go-server-project-scaffold`、`$go-server-rules-maintainer` |
| DotnetServer | `dotnet-server/agent.md` | .NET 服务端（Web API、gRPC、Worker） | `$dotnet-server-coding-guide`、`$dotnet-server-code-reviewer`、`$dotnet-server-project-scaffold`、`$dotnet-server-rules-maintainer` |
| PythonServer | `python-server/agent.md` | Python 服务端（FastAPI / Django / Flask） | `$python-server-coding-guide`、`$python-server-code-reviewer`、`$python-server-project-scaffold`、`$python-server-rules-maintainer` |
| JavaServer | `java-server/agent.md` | Java 服务端（Spring Boot / Spring Cloud） | `$java-server-coding-guide`、`$java-server-code-reviewer`、`$java-server-project-scaffold`、`$java-server-rules-maintainer` |
| NodeServer | `node-server/agent.md` | Node.js 服务端（NestJS / Express / Fastify） | `$node-server-coding-guide`、`$node-server-code-reviewer`、`$node-server-project-scaffold`、`$node-server-rules-maintainer` |
| Frontend | `frontend/agent.md` | 前端（admin / H5 / 小程序） | `$frontend-coding-guide`、`$frontend-code-reviewer`、`$frontend-project-scaffold`、`$frontend-rules-maintainer` |
| DotnetDesktop | `dotnet-desktop/agent.md` | .NET 桌面（WPF / MAUI / WinForms） | `$dotnet-desktop-coding-guide`、`$dotnet-desktop-code-reviewer`、`$dotnet-desktop-project-scaffold`、`$dotnet-desktop-rules-maintainer` |
| TauriDesktop | `tauri-desktop/agent.md` | Tauri 桌面（Rust + Tauri） | `$tauri-desktop-coding-guide`、`$tauri-desktop-code-reviewer`、`$tauri-desktop-project-scaffold`、`$tauri-desktop-rules-maintainer` |
| ElectronDesktop | `electron-desktop/agent.md` | Electron 桌面（Node.js + Chromium） | `$electron-desktop-coding-guide`、`$electron-desktop-code-reviewer`、`$electron-desktop-project-scaffold`、`$electron-desktop-rules-maintainer` |
| Android | `android/agent.md` | Android（Compose / XML Views） | `$android-coding-guide`、`$android-code-reviewer`、`$android-project-scaffold`、`$android-rules-maintainer` |
| iOS | `ios/agent.md` | iOS（SwiftUI / UIKit） | `$ios-coding-guide`、`$ios-code-reviewer`、`$ios-project-scaffold`、`$ios-rules-maintainer` |
| Flutter | `flutter/agent.md` | Flutter 跨平台（mobile / web / desktop） | `$flutter-coding-guide`、`$flutter-code-reviewer`、`$flutter-project-scaffold`、`$flutter-rules-maintainer` |
| ReactNative | `react-native/agent.md` | React Native 跨平台移动（iOS + Android） | `$react-native-coding-guide`、`$react-native-code-reviewer`、`$react-native-project-scaffold`、`$react-native-rules-maintainer` |
| Database | `database/agent.md` | 数据库 Schema 与迁移 | `$database-coding-guide`、`$database-code-reviewer`、`$database-project-scaffold`、`$database-rules-maintainer` |
| Collaboration | `collaboration/agent.md` | 前后端协作（契约 / 联调 / 发布） | `$frontend-backend-coding-guide`、`$frontend-backend-code-reviewer`、`$frontend-backend-rules-maintainer` |

## 协作协议

| 协议 | 文件 | 内容 |
|------|------|------|
| 协调协议 | `protocols/coordination.md` | 调度算法、执行顺序、上下文传递、冲突仲裁 |
| 交接协议 | `protocols/handoff.md` | Agent 间交接格式、触发条件、状态机 |
| 输出格式 | `protocols/agent-output-format.md` | 域 Agent 与 Coordinator 的标准输出格式 |
| 执行追溯 | `protocols/execution-trace.md` | 执行追溯摘要格式 |

## 平台适配

| 平台 | 适配文件 | 说明 |
|------|---------|------|
| OpenAI Codex | `adapters/codex/` | `.toml` 配置文件，可直接复制到 `.codex/agents/` |
| Cursor | `adapters/cursor/README.md` | 通过 `.cursor/rules/` + SKILL.md 实现 |
| Claude Code | `adapters/claude-code/README.md` | 通过 `CLAUDE.md` + agent teams 实现 |
| Gemini ADK | `adapters/gemini-adk/README.md` | 通过 Python ADK agent class 实现 |

## 与 rules/skills 的关系
1. Agent 模式是 rules + skills 之上的"编排层"，不替代任何现有文件。
2. 所有 agent.md 引用现有 skill 和 rules，不复制内容。
3. 跨域冲突仲裁规则以 `rules/index.md` 为准。
4. 域 Agent 内部执行 skill 时，skill 按自身的工作流运行，agent 层不干预 skill 内部逻辑。
5. **执行追溯（MUST）**：无论单体模式还是多 Agent 模式，每次任务执行完毕后必须附执行追溯摘要，告知用户调用链路（Agent / Skill / 加载规则 / 执行顺序 / 跨域交接）。格式定义见 `protocols/execution-trace.md`。

---
<!-- AI: 以下为目录结构参考，执行任务时无需加载，可在此停止读取 -->

## 目录结构
```
agents/
├── index.md                          ← 本文件（Agent 模式总入口）
├── _templates/
│   └── domain-agent-workflow.md     ← 域 Agent 通用工作流模板
├── protocols/
│   ├── coordination.md               ← 协调协议（全生命周期编排）
│   ├── handoff.md                    ← 交接协议
│   ├── agent-output-format.md       ← 输出格式
│   └── execution-trace.md           ← 执行追溯
├── coordinator/
│   └── agent.md                     ← Coordinator Agent
├── product/                          ← Product Agent（产品经理）
│   ├── agent.md                     ← Product Agent 定义
│   ├── phases/                      ← 五阶段引导问题模板
│   │   ├── 01-research.md           ← Phase 1：市场调研
│   │   ├── 02-competitive.md        ← Phase 2：竞品深度分析
│   │   ├── 03-requirements.md       ← Phase 3：需求定义
│   │   ├── 04-roadmap.md            ← Phase 4：MVP 与路线图
│   │   └── 05-summary.md           ← Phase 5：PRD 汇总
│   └── templates/                   ← PRD 输出模板
│       └── prd-template.md          ← PRD 文档模板
├── spec/                             ← Spec Agent（项目规格说明书）
│   ├── agent.md                     ← Spec Agent 定义
│   ├── phases/                      ← 五阶段引导问题模板
│   │   ├── 01-vision.md             ← Phase 1：项目愿景
│   │   ├── 02-decisions.md          ← Phase 2：技术决策
│   │   ├── 03-constraints.md        ← Phase 3：全局约束
│   │   ├── 04-modules.md            ← Phase 4：模块拆分
│   │   └── 05-summary.md           ← Phase 5：汇总输出
│   └── templates/                   ← Spec 输出模板
│       ├── project-spec-template.md ← 项目级 Spec 模板
│       └── module-spec-template.md  ← 模块级 Spec 模板
├── design/                           ← Design Agent（UI/UX 设计师）
│   ├── agent.md                     ← Design Agent 定义
│   ├── phases/                      ← 四阶段引导
│   │   ├── 01-research.md           ← Phase 1：设计调研
│   │   ├── 02-ux.md                 ← Phase 2：交互设计
│   │   ├── 03-ui.md                 ← Phase 3：视觉设计
│   │   └── 04-review.md            ← Phase 4：设计验证
├── go-server/
│   └── agent.md                     ← Go Server Agent
├── dotnet-server/
│   └── agent.md                     ← .NET Server Agent
├── python-server/
│   └── agent.md                     ← Python Server Agent
├── java-server/
│   └── agent.md                     ← Java Server Agent
├── node-server/
│   └── agent.md                     ← Node.js Server Agent
├── frontend/
│   └── agent.md                     ← Frontend Agent
├── dotnet-desktop/
│   └── agent.md                     ← .NET Desktop Agent
├── tauri-desktop/
│   └── agent.md                     ← Tauri Desktop Agent
├── electron-desktop/
│   └── agent.md                     ← Electron Desktop Agent
├── android/
│   └── agent.md                     ← Android Agent
├── ios/
│   └── agent.md                     ← iOS Agent
├── flutter/
│   └── agent.md                     ← Flutter Agent
├── react-native/
│   └── agent.md                     ← React Native Agent
├── database/
│   └── agent.md                     ← Database Agent
├── collaboration/
│   └── agent.md                     ← Collaboration Agent
├── security/
│   └── agent.md                     ← Security Agent（安全工程师）
├── qa/
│   └── agent.md                     ← QA Agent（质量保障）
├── devops/
│   └── agent.md                     ← DevOps Agent（部署运维）
└── adapters/
    ├── mcp-tools.md                 ← 跨平台 MCP 工具方案
    ├── codex/                       ← OpenAI Codex 适配
    ├── cursor/                      ← Cursor 适配
    ├── claude-code/                 ← Claude Code 适配
    └── gemini-adk/                  ← Gemini ADK 适配
```
