# 域 Agent 通用工作流

本文件定义所有域 Agent（技术栈域）的共享工作流。各域 `agent.md` 通过 `workflow` 字段引用本模板，并声明域参数。

## 标准工作流

### 0. 文件定位与验证（subagent 模式 MUST 执行）

域 Agent 以 subagent 形式运行时，不具备主 AI 的 `available_skills` 上下文，必须先定位所需文件。

**从调度请求中提取文件路径：**
- 调度请求的"文件路径引用"部分包含：域 Agent 定义、Skill 入口文件、工作流模板、场景路由表、规则索引。
- 按以下顺序依次读取文件：
  1. **域 Agent 定义**（`agents/<domain>/agent.md`）→ 了解职责边界和可用 Skill 清单。
  2. **Skill 入口文件**（`skills/<skill-name>/SKILL.md`）→ 获取域参数（`baseline_files`、`scenario_map`、`rules_index`、`max_load`）。
  3. **场景路由表**（`skills/<skill-name>/references/coding-scenario-map.md`）→ 匹配编码场景，确定规则加载集。
  4. **规则索引**（`rules/<domain>/index.md`）→ 验证规则文件存在。

**降级路径约定（当调度请求缺少文件路径时）：**
- 域 Agent 定义：`agents/<domain>/agent.md`（`<domain>` 从调度请求的 Agent 名称推断，如 Database → `database`、GoServer → `go-server`、Frontend → `frontend`）。
- Skill 入口文件：`skills/<domain>-coding-guide/SKILL.md`（coding 类任务）；其他类型替换后缀（`-code-reviewer`、`-project-scaffold`、`-rules-maintainer`）。
- 工作流模板：`skills/_templates/coding-guide-workflow.md`（coding 类任务）。
- 场景路由表：`skills/<domain>-coding-guide/references/coding-scenario-map.md`。
- 规则索引：`rules/<domain>/index.md`。
- 特殊域映射：Collaboration → `skills/frontend-backend-coding-guide/`、`rules/frontend-backend-collaboration.md`。

**验证：** 如果关键文件不可达（读取失败），在执行追溯中标记为"缺失"并说明原因，避免静默跳过。

### 1. 接收调度请求
- 从 Coordinator 接收任务摘要、任务类型、文件路径引用、域上下文。
- 按域 agent.md 声明的 `context`（如部署模式、应用类型、框架）确定当前上下文。

### 2. 选择 Skill 并执行
- 根据任务类型从域 agent.md 声明的 `skills` 表中选择对应 skill。
- 从调度请求的文件路径引用中获取 Skill 入口文件路径，读取 SKILL.md。
- Skill 内部按场景路由表加载规则并执行。

### 3. 跨域交接
- 按 `agents/protocols/handoff.md` 定义的交接协议，结合域 agent.md 声明的交接规则执行：
  - 需要数据库 Schema 变更 → 交接给 Database Agent。
  - 需要 API 契约变更 → 交接给 Collaboration Agent。
  - 规则冲突 → 上报 Coordinator Agent。

### 4. 输出报告
- 按 `agents/protocols/agent-output-format.md` 格式输出执行报告。
- 按 `agents/protocols/execution-trace.md` 格式附执行追溯。
