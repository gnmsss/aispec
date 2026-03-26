# agents/protocols/coordination.md — 协调协议

## 目的
1. 定义 Coordinator Agent 如何接收任务、识别域、调度域 Agent 的标准流程。
2. 确保多 Agent 模式下的执行顺序、并行策略、上下文传递有统一规范。

## 适用范围
1. 仅在多 Agent 模式下生效。
2. 单体模式（单个 AI 实例直接调用 `$skill-name`）不受本协议约束。

## 调度算法

### 1. 域识别
- Coordinator 复用 `skills/task-router/references/domain-detect-map.md` 的关键词映射表。
- 新增产品、设计、测试、运维域的关键词识别（见下文任务类型识别）。
- 从用户任务描述中提取关键词，匹配到涉及的域。
- 若无法确定，向用户询问确认。

### 2. 执行顺序 — 全生命周期
- 默认顺序（按产品开发生命周期）：

| 阶段 | 域 | 说明 |
|------|-----|------|
| -1 | Product | 产品需求定义：竞品分析、PRD（新产品/重大迭代时执行，已有 PRD 则跳过） |
| 0 | Spec | 技术规格定义：架构、选型、模块拆分（新项目/重大迭代时执行，已有 Spec 则跳过） |
| 0.5 | Design | UI/UX 设计：交互流程、视觉设计、设计系统（可与 Spec 并行） |
| 0.8 | **Task Planning** | Coordinator 调用 `$task-planner` 将 Spec + Design 产出拆解为任务清单（Spec 和 Design 均完成后执行） |
| 1 | Database | 数据库 Schema 与迁移必须先行 |
| 2 | GoServer / DotnetServer / PythonServer / JavaServer / NodeServer | 服务端 API 基于数据库结构开发 |
| 3 | Collaboration | 前后端契约对齐，确保接口一致性 |
| 4 | Frontend / DotnetDesktop / TauriDesktop / ElectronDesktop / Android / iOS / Flutter / ReactNative | 客户端基于 Design 设计稿 + API 契约开发 |
| 4.5 | Security | 安全审计：威胁建模、OWASP 检查、依赖扫描（开发阶段完成后执行） |
| 5 | QA | 系统测试、验收测试（安全审计通过后执行） |
| 6 | DevOps | 部署上线、监控配置（测试通过后执行） |

- 同阶段内的域 Agent 可并行执行（如 Android 和 iOS 可并行）。
- Spec（阶段 0）和 Design（阶段 0.5）可并行执行，互不阻塞。
- 仅涉及单域的任务直接调度对应域 Agent，无需经过全部阶段。
- 阶段 -1（Product）和阶段 5-6（QA/DevOps）仅在完整产品流程中触发，日常编码任务可跳过。

### 3. MCP 可用性前置检查
在调度涉及外部 MCP 的 Agent 之前，Coordinator 必须检查所需 MCP 是否可用：

- 调度 Product Agent → 检查搜索 MCP（必需）和 Playwright MCP（可选）。
- 调度 Design Agent → 检查 Pencil MCP（必需）、搜索 MCP（可选）和 Playwright MCP（可选）。
- 调度 QA Agent → 检查 Playwright MCP（可选）。
- 调度 Security Agent → 无需检查（安全审计不依赖外部 MCP）。
- 调度其他域 Agent → 无需检查。

检查不通过时，按 `agents/adapters/mcp-tools.md` 中的"MCP 可用性检查与安装引导协议"执行（告知→询问→安装/退化）。所有 MCP 状态确认后，再正式调度 Agent 执行任务。

### 4. 并行/串行决策规则
- 同一阶段内无数据依赖的域 Agent → 并行。
- 跨阶段有数据依赖的域 Agent → 串行（前一阶段完成后再启动下一阶段）。
- Coordinator 标注每个域 Agent 的依赖关系，域 Agent 不自行决定执行顺序。

## 上下文传递

### Subagent 上下文限制（MUST 理解）

在 Cursor 等平台的多 Agent 模式下，域 Agent 通常以 **subagent**（如 Cursor Task subagent）形式 spawn 执行。Subagent 与主 AI 的关键差异：

| 能力 | 主 AI（Coordinator） | Subagent（域 Agent） |
|------|---------------------|---------------------|
| `available_skills` 上下文 | 有（系统提示中列出所有 Skill 的绝对路径） | **无**（仅接收 prompt 文本） |
| 自动定位 SKILL.md | 可以 | **不可以** |
| 读取工作区文件 | 可以 | 可以（但需知道路径） |

因此 Coordinator 在调度请求中**必须提供完整的文件路径引用**，使 subagent 能定位 Agent 定义、Skill 文件、场景路由表和规则文件。路径省略是域 Agent 规则加载失败的首要原因。

### 调度请求格式（Coordinator → 域 Agent）
Coordinator 向每个域 Agent 发送调度请求时，必须包含以下信息：

```
## 调度请求

### 任务信息
- 任务摘要：{用户原始任务的精简描述}
- 任务类型：{product / spec / design / design-review / coding / review / scaffold / rule-maintenance / security-audit / testing / devops}
- 任务 ID：{$task-planner 生成的任务 ID，如 ORDER-DB-001；无则留空}

### 文件路径引用（MUST — subagent 依赖这些路径定位文件）
- 域 Agent 定义：{agents/<domain>/agent.md}
- Skill 入口文件：{skills/<skill-name>/SKILL.md}
- 工作流模板：{skills/_templates/<workflow>.md，从 SKILL.md 的 workflow 字段获取}
- 场景路由表：{skills/<skill-name>/references/coding-scenario-map.md，从 SKILL.md 的 scenario_map 字段获取}
- 规则索引：{rules/<domain>/index.md，从 SKILL.md 的 rules_index 字段获取}

### 域上下文
- 前序输出：{前序域 Agent 的输出摘要，如 Schema 定义、API 契约等}
- Spec 输入：{从 Spec 中截取的相关章节内容}
- Design 输入：{从设计稿中截取的相关页面/组件信息，无则留空}
- 约束提示：{跨域冲突仲裁提示，如"数据库规则优先级最高"}
```

#### 文件路径查表方法
Coordinator 从 `agents/index.md` 的"Agent 索引"表中获取每个域 Agent 的定义文件和可用 Skill。然后从对应的 SKILL.md 中获取 `workflow`、`scenario_map`、`rules_index` 字段值。常用域的文件路径速查：

| 域 | Agent 定义 | Skill（coding） | 场景路由表 | 规则索引 |
|----|-----------|----------------|-----------|---------|
| Database | `agents/database/agent.md` | `skills/database-coding-guide/SKILL.md` | `skills/database-coding-guide/references/coding-scenario-map.md` | `rules/database/index.md` |
| GoServer | `agents/go-server/agent.md` | `skills/go-server-coding-guide/SKILL.md` | `skills/go-server-coding-guide/references/coding-scenario-map.md` | `rules/go-server/index.md` |
| Frontend | `agents/frontend/agent.md` | `skills/frontend-coding-guide/SKILL.md` | `skills/frontend-coding-guide/references/coding-scenario-map.md` | `rules/frontend/index.md` |
| Collaboration | `agents/collaboration/agent.md` | `skills/frontend-backend-coding-guide/SKILL.md` | `skills/frontend-backend-coding-guide/references/coding-scenario-map.md` | `rules/frontend-backend-collaboration.md` |
| DotnetServer | `agents/dotnet-server/agent.md` | `skills/dotnet-server-coding-guide/SKILL.md` | `skills/dotnet-server-coding-guide/references/coding-scenario-map.md` | `rules/dotnet-server/index.md` |
| PythonServer | `agents/python-server/agent.md` | `skills/python-server-coding-guide/SKILL.md` | `skills/python-server-coding-guide/references/coding-scenario-map.md` | `rules/python-server/index.md` |
| JavaServer | `agents/java-server/agent.md` | `skills/java-server-coding-guide/SKILL.md` | `skills/java-server-coding-guide/references/coding-scenario-map.md` | `rules/java-server/index.md` |
| NodeServer | `agents/node-server/agent.md` | `skills/node-server-coding-guide/SKILL.md` | `skills/node-server-coding-guide/references/coding-scenario-map.md` | `rules/node-server/index.md` |
| Android | `agents/android/agent.md` | `skills/android-coding-guide/SKILL.md` | `skills/android-coding-guide/references/coding-scenario-map.md` | `rules/android/index.md` |
| iOS | `agents/ios/agent.md` | `skills/ios-coding-guide/SKILL.md` | `skills/ios-coding-guide/references/coding-scenario-map.md` | `rules/ios/index.md` |
| Flutter | `agents/flutter/agent.md` | `skills/flutter-coding-guide/SKILL.md` | `skills/flutter-coding-guide/references/coding-scenario-map.md` | `rules/flutter/index.md` |
| ReactNative | `agents/react-native/agent.md` | `skills/react-native-coding-guide/SKILL.md` | `skills/react-native-coding-guide/references/coding-scenario-map.md` | `rules/react-native/index.md` |
| DotnetDesktop | `agents/dotnet-desktop/agent.md` | `skills/dotnet-desktop-coding-guide/SKILL.md` | `skills/dotnet-desktop-coding-guide/references/coding-scenario-map.md` | `rules/dotnet-desktop/index.md` |
| TauriDesktop | `agents/tauri-desktop/agent.md` | `skills/tauri-desktop-coding-guide/SKILL.md` | `skills/tauri-desktop-coding-guide/references/coding-scenario-map.md` | `rules/tauri-desktop/index.md` |
| ElectronDesktop | `agents/electron-desktop/agent.md` | `skills/electron-desktop-coding-guide/SKILL.md` | `skills/electron-desktop-coding-guide/references/coding-scenario-map.md` | `rules/electron-desktop/index.md` |

注：非 coding 类任务（review / scaffold / rule-maintenance）替换对应的 skill 名称即可（如 `*-code-reviewer`、`*-project-scaffold`、`*-rules-maintainer`）。Product / Spec / Design / Security / QA / DevOps 域的 Skill 路径参见 `agents/index.md` 的 Agent 索引表。

### 结果回传格式（域 Agent → Coordinator）
遵循 `agents/protocols/agent-output-format.md` 的标准输出格式，按 `agents/protocols/execution-trace.md` 格式附执行追溯。

## 跨域冲突仲裁
- 多 Agent 模式下的跨域冲突仲裁规则与单体模式一致，以 `rules/index.md` 的仲裁规则为准。
- Coordinator 在汇总各域 Agent 结果时执行冲突检测：
  1. 数据库相关冲突 → 以 Database Agent 的输出为准。
  2. 前后端接口冲突 → 以 Collaboration Agent 的输出为准。
  3. 域内冲突 → 以"更严格且可验证"的条款为准。
- 若冲突无法自动消解，Coordinator 向用户报告冲突并请求裁定。

## 任务类型识别
Coordinator 根据用户意图识别任务类型，决定调度哪个 Agent 及其 skill：

| 用户意图 | 任务类型 | 域 Agent 调用的 skill |
|---------|---------|---------------------|
| 产品需求/竞品分析/PRD | product | Product Agent 的 `$product-prd-writer` |
| 新项目立项/技术方案设计/规格定义 | spec | Spec Agent 的 `$spec-generator` |
| UI/UX 设计/界面原型/设计系统 | design | Design Agent 的 `$ui-ux-designer` |
| 设计走查/设计还原度审查 | design-review | Design Agent 的 `$design-reviewer` |
| 编写/修改业务代码 | coding | `*-coding-guide` |
| 审查代码变更 / PR 评审 | review | `*-code-reviewer` |
| 初始化新项目 | scaffold | `*-project-scaffold` |
| 新增/修改/审计规则 | rule-maintenance | `*-rules-maintainer` |
| 安全审计/威胁建模/漏洞扫描/安全合规 | security-audit | Security Agent 的 `$security-auditor` |
| 测试策略/测试用例/验收测试 | testing | QA Agent 的 `$qa-test-strategist` |
| CI/CD/部署/监控/基础设施 | devops | DevOps Agent 的 `$devops-engineer` |

注：
- product 类型任务由 Product Agent 独立执行（Phase -1），产出 PRD 作为后续 Agent 输入。
- spec 类型任务由 Spec Agent 独立执行（Phase 0），产出 Spec 作为后续 Agent 输入。
- design 类型任务由 Design Agent 执行（Phase 0.5），可与 Spec 并行。
- security-audit 类型任务由 Security Agent 在开发完成后执行（Phase 4.5）。
- testing 类型任务由 QA Agent 在安全审计通过后执行（Phase 5）。
- devops 类型任务由 DevOps Agent 在测试通过后执行（Phase 6）。
- 完整产品流程：Product → Spec ∥ Design → Task Planning → Database → Server → Collaboration → Client → Security → QA → DevOps。
- Task Planning（阶段 0.8）由 Coordinator 调用 `$task-planner` 自动执行，将 Spec + Design 产出拆解为按域分组的可执行任务清单。

## 汇总输出
- Coordinator 收集所有域 Agent 的结果后，按 `agents/protocols/agent-output-format.md` 的汇总格式输出，按 `agents/protocols/execution-trace.md` 格式附执行追溯。
- 汇总内容包括：各域执行结果、跨域依赖关系、冲突处理记录、整体规则清单。
