# Cursor 适配说明

## 概述
Cursor（v2.4+）通过 `.cursor/rules/`、`SKILL.md` 和内建 subagent 机制支持多 Agent 工作流。
本文档说明如何在 Cursor 中使用本仓库的 Agent 模式。

## 机制对应关系

| Agent 模式概念 | Cursor 对应机制 |
|---------------|---------------|
| Agent 定义（agent.md） | `.cursor/rules/` 规则文件（将 agent.md 作为规则加载） |
| Skill（SKILL.md） | Cursor Skills（原生支持，直接引用 `skills/*/SKILL.md`） |
| 域 Agent 并行执行 | Cursor Task subagent（`generalPurpose` / `explore`） |
| Coordinator 调度 | 主对话中 AI 充当 Coordinator，通过 Task 工具 spawn subagent |

## 配置步骤

### 1. 单体模式（默认，无需额外配置）
Cursor 原生支持 SKILL.md。只需确保 `skills/` 目录在项目中，AI 会自动发现并使用 skill。

### 2. 多 Agent 模式

#### 方式 A：手动触发（推荐）
在 Cursor 对话中直接指示 AI 使用多 Agent 模式：

```
请使用多 Agent 模式完成以下任务。
参考 agents/index.md 了解 Agent 体系，
参考 agents/protocols/coordination.md 了解调度协议。
任务：新增一个用户管理功能，包含数据库、API 和前端页面。
```

AI 会读取 Agent 定义文件，理解各 Agent 的职责边界，并通过 Task subagent 并行执行。

#### 方式 B：创建规则文件
将 Agent 模式的入口信息写入 `.cursor/rules/`：

```bash
# 在项目 .cursor/rules/ 下创建规则文件
```

`.cursor/rules/agent-mode.md` 内容示例：
```markdown
---
description: 当用户要求使用多 Agent 模式或提出跨域任务时加载
globs: []
alwaysApply: false
---
多 Agent 模式已启用。请读取 agents/index.md 了解 Agent 体系。
按 agents/protocols/coordination.md 的调度协议执行跨域任务。
每个域 Agent 的定义在 agents/<domain>/agent.md 中。
```

### 3. 并行执行
Cursor 的 Task subagent 支持并行执行。AI 可以：
- 为每个域 Agent 创建一个 Task subagent。
- 每个 subagent 加载对应的 `agents/<domain>/agent.md` 和 `skills/*/SKILL.md`。
- 主对话中的 AI 充当 Coordinator，汇总各 subagent 的结果。

### 4. Subagent Prompt 构造（MUST）

#### 核心问题
Cursor Task subagent **不具备** 主 AI 的 `available_skills` 上下文。主 AI 的系统提示中列出了所有 Skill 的 SKILL.md 绝对路径，subagent 没有这些信息。如果 Coordinator 在 Task prompt 中不提供完整的文件路径，subagent 将无法定位 Skill 文件、场景路由表和规则文件，导致规则加载失败。

#### Prompt 构造模板
Coordinator 通过 Task 工具 spawn 域 Agent subagent 时，prompt 必须包含以下结构：

```
你是 {Agent 名称}（{角色描述}）。

## 初始化步骤（按顺序执行）
1. 阅读你的 Agent 定义文件：{agents/<domain>/agent.md}
2. 根据任务类型，阅读对应的 Skill 入口文件：{skills/<skill-name>/SKILL.md}
3. 阅读 Skill 引用的工作流模板：{skills/_templates/<workflow>.md}
4. 阅读场景路由表：{skills/<skill-name>/references/coding-scenario-map.md}
5. 从场景路由表中匹配当前编码场景，确定需要加载的规则文件
6. 按规则文件路径（以 rules/ 为前缀，相对于项目根目录）读取规则

## 调度请求
{按 agents/protocols/coordination.md 的调度请求格式填写}

## 输出要求
- 按 agents/protocols/agent-output-format.md 格式输出执行报告
- 按 agents/protocols/execution-trace.md 格式附执行追溯
```

#### 完整示例：Database Agent Subagent Prompt

```
你是 Database Agent（数据库领域专家），负责数据库 Schema 初始化和迁移脚本编写。

## 初始化步骤（按顺序执行）
1. 阅读你的 Agent 定义文件：agents/database/agent.md
2. 阅读 Skill 入口文件：skills/database-coding-guide/SKILL.md
3. 阅读工作流模板：skills/_templates/coding-guide-workflow.md
4. 阅读场景路由表：skills/database-coding-guide/references/coding-scenario-map.md
5. 从场景路由表中匹配当前编码场景，确定需要加载的规则文件
6. 按规则文件路径读取规则（如 rules/database/database.md）

## 调度请求

### 任务信息
- 任务摘要：新增订单管理功能的数据库表结构
- 任务类型：coding
- 任务 ID：ORDER-DB-001

### 文件路径引用
- 域 Agent 定义：agents/database/agent.md
- Skill 入口文件：skills/database-coding-guide/SKILL.md
- 工作流模板：skills/_templates/coding-guide-workflow.md
- 场景路由表：skills/database-coding-guide/references/coding-scenario-map.md
- 规则索引：rules/database/index.md

### 域上下文
- 前序输出：无（Database 是首个执行的域）
- Spec 输入：订单表需包含 order_id, user_id, total_amount, status, created_at 字段
- Design 输入：无
- 约束提示：数据库规则在跨域冲突中拥有最高优先级

## 输出要求
- 按 agents/protocols/agent-output-format.md 格式输出执行报告
- 按 agents/protocols/execution-trace.md 格式附执行追溯
```

#### 常见错误
| 错误做法 | 后果 | 正确做法 |
|---------|------|---------|
| prompt 中只写"使用 $database-coding-guide" | subagent 不知道 SKILL.md 路径，无法加载 | 提供完整路径 `skills/database-coding-guide/SKILL.md` |
| 省略场景路由表路径 | subagent 无法匹配编码场景 | 提供 `skills/*/references/coding-scenario-map.md` |
| 只写规则文件名（如 `database.md`） | subagent 不知道文件在哪个目录 | 写完整路径 `rules/database/database.md` |
| 不提供初始化步骤 | subagent 不知道阅读文件的顺序 | 按模板提供按序初始化步骤 |

## 使用示例

### 项目规格定义
```
使用多 Agent 模式。参考 agents/index.md。
我要做一个电商平台，请使用 Spec Agent 引导我完成技术规格定义。
参考 agents/spec/agent.md 了解五阶段引导流程。
```

### 跨域编码
```
使用多 Agent 模式。参考 agents/index.md 和 agents/protocols/coordination.md。
任务：新增订单管理功能（数据库 + Go 后端 + 前端页面）。
请按 数据库→服务端→前后端协作→前端 的顺序执行。
```

### 并行代码审查
```
使用多 Agent 模式审查当前分支的代码变更。
为每个涉及的技术域创建一个 subagent 并行审查。
审查规则参考 agents/<domain>/agent.md 中的 code-reviewer skill。
```

## 限制
1. Cursor subagent 不支持自定义 model 配置（使用全局模型设置）。
2. subagent 间不能直接通信，需通过主对话传递上下文。
3. 适合 2-4 个并行 subagent，过多会增加 token 消耗。
