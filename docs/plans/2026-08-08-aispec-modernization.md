# AISpec 现代化改造实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 AISpec 从「15 域 × 4 类 skill + 自建 agent 编排」重构为「AGENTS.md 入口 + 10 域精简规则 + 15 个标准 SKILL.md」,内容更新至 2026-08 技术现状。

**Architecture:** 规则层每域压缩为 baseline.md(硬约束+红线)+ practices.md(场景实践);技能层每域 1 个三合一 skill;AGENTS.md 取代 rules/index.md 与全部编排文档。spec 见 `docs/specs/2026-08-08-aispec-modernization-design.md`。

**Tech Stack:** 纯 Markdown 内容工程;git;PowerShell/Bash 校验命令。

## Global Constraints

- 保留域:go-server、dotnet-server、python-server、node-server、frontend、tauri-desktop、dotnet-desktop、flutter、android、ios、database
- 删除域:java-server、electron-desktop、react-native
- Skill 标准:frontmatter 仅 `name`(与目录名一致)+ `description`(含触发场景);禁止 `workflow:` 字段、`$skill` 引用、执行追溯要求
- 压缩筛选标准:只保留可验证硬约束与决策性指引;删除描述性废话、重复条款、面向人类流程的模板(评审表、申请单)
- 版本基线(2026-08,重写时不确定的联网核实):Go 1.25+ / .NET 10 LTS / Python 3.12+ + uv + ruff + Pydantic v2 / Node.js 22+ LTS + ESM / Vue 3.5+ + Vite + Tailwind v4、React 19 / Tauri 2.x / Kotlin 2.x + Compose + targetSdk 36 / Swift 6 + SwiftUI + Xcode 26 / Flutter 最新稳定版(核实)
- 每域 baseline.md ≤ 150 行,practices.md ≤ 300 行;跨域单文件 ≤ 250 行
- 所有文件路径引用必须指向仓库内实际存在的文件(最终校验把关)
- 每个任务独立 commit,消息用中文,格式 `refactor|feat|docs: <描述>`

---

### Task 1: 清除废弃层(agents、废弃域、外围)

**Files:**
- Delete: `agents/`(全部)、`.cursor/`
- Delete: `rules/java-server/`、`rules/java-server.md`、`rules/electron-desktop/`、`rules/electron-desktop.md`、`rules/react-native/`、`rules/react-native.md`
- Delete: `rules/templates/`、`rules/monorepo/`、`rules/i18n/`、`rules/release/`
- Delete: `skills/java-server-*`、`skills/electron-desktop-*`、`skills/react-native-*`(各 4 个)
- Delete: `skills/task-router/`、`skills/task-planner/`、`skills/product-prd-writer/`、`skills/qa-test-strategist/`、`skills/devops-engineer/`、`skills/frontend-backend-*`(3 个)、`skills/_templates/`

- [ ] **Step 1: git rm 上述目录与文件**(`git rm -r -q agents .cursor rules/java-server ...`)
- [ ] **Step 2: 校验**:`ls rules skills` 确认无残留;`git status` 只含删除
- [ ] **Step 3: Commit** `refactor: 删除 agent 编排层、废弃技术域与外围模板`

### Task 2~11: 逐域压缩规则 + 重写域 skill(×10)

依序处理:go-server、dotnet-server、python-server、node-server、frontend、tauri-desktop、dotnet-desktop、flutter、android、ios。每域执行:

**Files(以 go-server 为例,其余域同构):**
- Read: `rules/go-server/index.md`、`rules/go-server/common/*.md`、`rules/go-server/profiles/*`
- Create: `rules/go-server/baseline.md`(≤150 行:红线 MUST NOT + 硬约束 MUST,合并原 baseline/forbidden/code-style/error-handling 核心)
- Create: `rules/go-server/practices.md`(≤300 行:API 设计、数据访问、并发/资源、配置、可观测性要点、测试;profiles 差异以小节合入)
- Delete: `rules/go-server/index.md`、`common/`、`profiles/`、`rules/go-server.md`(兼容入口)
- Delete: `skills/go-server-coding-guide/`、`skills/go-server-code-reviewer/`、`skills/go-server-project-scaffold/`、`skills/go-server-rules-maintainer/`
- Create: `skills/go-server/SKILL.md`(三节:编码引导→何时读 baseline/practices;审查清单→10~20 条可执行检查;脚手架→目录结构与初始化要点。跨域联动直接写 rules 文件路径,如 `rules/collaboration.md`、`rules/database.md`)

- [ ] **Step 1: 通读旧域规则,按筛选标准提取条款,更新到该域 2026-08 版本基线**(过时实践删除或改写;不确定版本联网核实)
- [ ] **Step 2: 写入 baseline.md + practices.md,行数校验** `wc -l rules/<域>/*.md`
- [ ] **Step 3: 删除旧规则文件与旧 4 类 skill,写入新 SKILL.md**
- [ ] **Step 4: 校验**:`ls rules/<域>` 仅 2 文件;SKILL.md frontmatter 仅 name/description 且 name==目录名;grep 无 `$` skill 引用、无 `agents/` 引用
- [ ] **Step 5: Commit** `refactor(<域>): 压缩规则至 baseline+practices,合并 4 类 skill 为域 skill`

### Task 12: 跨域规范压缩

**Files:**
- Create: `rules/database.md`(源:`rules/database/index.md|database.md|data-migration.md`)
- Create: `rules/security.md`(源:`rules/security/security-baseline.md`)
- Create: `rules/collaboration.md`(源:`rules/frontend-backend-collaboration.md` + `rules/api-versioning/api-versioning.md`)
- Create: `rules/design.md`(源:`rules/design/` 6 文件)
- Create: `rules/testing.md`(源:`rules/testing/` 2 文件)
- Create: `rules/operations.md`(源:`rules/observability/observability.md` + `rules/environment/environment-management.md`)
- Delete: 上述全部源目录/文件 + `rules/index.md`(职责移交 AGENTS.md)
- Create: `skills/database/SKILL.md`(schema 设计、迁移脚本、审查清单)

- [ ] **Step 1: 逐文件压缩写入(≤250 行/文件),内容按 2026 现状更新**
- [ ] **Step 2: 删除源文件;校验 `ls rules/` 结构 == 设计目标;行数校验**
- [ ] **Step 3: Commit** `refactor(rules): 跨域规范压缩为 6 个单文件`

### Task 13: 横切 skills 重写

**Files:**
- Rewrite: `skills/spec-generator/SKILL.md`(去 agents/protocols 依赖、去执行追溯;保留五阶段引导主体)
- Rewrite: `skills/security-auditor/SKILL.md`(规则引用改为 `rules/security.md`)
- Create: `skills/design-guide/SKILL.md`(合并 `ui-ux-designer` + `design-reviewer`,规则引用 `rules/design.md`)
- Create: `skills/rules-maintainer/SKILL.md`(通用版:校验任意域规则文件结构/行数/引用完整性)
- Delete: `skills/ui-ux-designer/`、`skills/design-reviewer/`、其余未处理旧 skill 残留

- [ ] **Step 1: 重写/创建 4 个 skill;各自 references/ 仅保留仍被正文引用的文件**
- [ ] **Step 2: 校验 `ls skills/` == 15 个目录;frontmatter 校验同 Task 2**
- [ ] **Step 3: Commit** `refactor(skills): 横切 skill 重写为标准格式`

### Task 14: 入口与文档重写

**Files:**
- Create: `AGENTS.md`(≤150 行:项目定位一句话;域索引表(域→rules 路径→skill 名);加载策略(先 baseline 后按需 practices,跨域引用 collaboration/database);冲突仲裁精简版(database > security > collaboration > 跨域 > 域内);无"模式"概念)
- Create: `CLAUDE.md`(内容仅 `@AGENTS.md` 一行 + 一句说明)
- Rewrite: `README.md`(定位、结构图、四 CLI 安装表(按设计文档"安装模型"节)、版本基线声明;删除 star 营销、agent 相关全部内容)
- Modify: `CONTRIBUTING.md`(删除 agent 贡献规范章节,skill/rules 规范对齐新结构)

- [ ] **Step 1: 写入 4 个文件**
- [ ] **Step 2: Commit** `docs: AGENTS.md 入口 + README 四 CLI 安装指南`

### Task 15: 全局一致性校验

- [ ] **Step 1: 死引用扫描**:`grep -rn "agents/\|java-server\|electron-desktop\|react-native\|task-router\|task-planner\|execution-trace\|rules/index.md\|\$[a-z-]*-guide\|templates/" --include="*.md" .`(排除 docs/、node_modules/、.git);逐条修复
- [ ] **Step 2: 路径存在性校验**:提取所有 md 中 `rules/...` `skills/...` 引用,逐一 Test-Path
- [ ] **Step 3: 统计核对**:rules *.md ≈ 26,skills 目录 == 15,agents 不存在
- [ ] **Step 4: Commit** `chore: 一致性校验修复`
