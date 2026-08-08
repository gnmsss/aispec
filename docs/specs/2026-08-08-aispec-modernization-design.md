# AISpec 现代化改造设计(2026-08)

## 背景与动机

AISpec 建于 2026 年 3 月,包含 341 个规则文件(2.5MB)、68 个 skill、22 个 agent + 自建协作协议 + 4 平台适配器。半年后的现状:

1. **Agent 编排层过时**:Claude Code(subagents)、Codex、Grok Build(multi-agent)、OpenCode(agents)均已内置原生 agent 机制,自建 Coordinator/协议/adapter 无存在价值。
2. **Skill 格式收敛**:Anthropic SKILL.md 格式已成为事实标准,四大 CLI 全部支持(Codex 读 `.agents/skills/`,OpenCode 兼容读 `.claude/skills` 与 `.agents/skills`,Grok Build 读项目 skills 目录与 `~/.grok/skills/`);AGENTS.md 成为项目规范标准入口(Claude Code 通过 `@AGENTS.md` 导入兼容)。
3. **规则臃肿**:15 域 × 4 类 skill 的笛卡尔积 + 352KB 模板,加载策略复杂,部分技术域已不再使用,部分内容(框架版本、实践)已过时。

## 目标平台

Claude Code、OpenAI Codex CLI、Grok Build CLI、OpenCode——全部为 CLI 形态。

## 决策(已与维护者确认)

| 决策项 | 结论 |
|--------|------|
| agents/ 目录 | **彻底删除**(含协议、adapters) |
| 入口文件 | **AGENTS.md** 为唯一入口;`CLAUDE.md` 仅一行 `@AGENTS.md` 导入 |
| Skill 粒度 | **按域合并**:每域 1 个 skill(编码引导+审查+脚手架三合一) |
| 删除的技术域 | java-server、electron-desktop、react-native |
| 保留的技术域 | go-server、dotnet-server、python-server、node-server、frontend、tauri-desktop、dotnet-desktop、flutter、android、ios、database |
| 跨域规范 | 保留并压缩:security、collaboration(并入 api-versioning)、design、testing;observability+environment 合并为 operations.md;删除 release、i18n、monorepo、templates/ |
| 横切 skill | 保留 spec-generator、security-auditor、design-guide(ui-ux-designer 与 design-reviewer 合并)、rules-maintainer(15 个合并为 1 个通用) |
| 内容更新 | 所有保留内容更新至 2026 年技术现状(框架版本、工具链、实践) |

## 目标结构

```
aispec/
├── AGENTS.md                  ← 唯一入口:域索引 + 加载策略 + 冲突仲裁(精简)
├── CLAUDE.md                  ← @AGENTS.md 导入(Claude Code 兼容)
├── README.md                  ← 重写:定位 + 4 CLI 安装指南
├── rules/
│   ├── go-server/             ← 每域 2 文件
│   │   ├── baseline.md        ← 必载:硬约束 + 红线(合并原 baseline+forbidden+核心条款)
│   │   └── practices.md       ← 按需:场景实践(API/数据库访问/并发/缓存/测试,profiles 择要合入)
│   ├── dotnet-server/ …       ← 同构 ×10 域
│   ├── database.md
│   ├── security.md
│   ├── collaboration.md       ← 前后端协作 + API 契约 + API 版本
│   ├── design.md              ← 原 design/ 6 文件压缩
│   ├── testing.md             ← e2e + performance 压缩
│   └── operations.md          ← observability + environment 精简合并
└── skills/
    ├── go-server/SKILL.md …   ← 10 个域 skill(标准格式,无自定义字段)
    ├── database/SKILL.md
    ├── spec-generator/
    ├── security-auditor/
    ├── design-guide/
    └── rules-maintainer/
```

规模预期:规则文件 341 → 约 26 个;skills 68 → 15 个;agents 72 文件 → 0。

## Skill 规范

- 标准 frontmatter:仅 `name` + `description`(name 与目录名一致,description 含触发场景)。
- 移除自定义 `workflow:` 字段、`$skill-name` 引用语法、执行追溯协议(依赖已删除的 agents/protocols)。
- 域 skill 正文三节:编码引导(何时载 baseline/practices)、代码审查清单、脚手架要点;跨域联动改为直接引用 rules 文件路径。
- 复杂参考材料放 `references/`(仅在确有必要时保留)。

## 内容更新基线(2026-08)

各域规则重写时对齐当前主流版本与实践,包括但不限于:Go 1.25+、.NET 10 (LTS)、Python 3.12+(uv/pydantic v2)、Node.js 22+ LTS(ESM 优先)、Vue 3.5+/React 19/Vite/Tailwind v4、Tauri 2.x、Kotlin 2.x + Compose、Swift 6 + SwiftUI、Flutter 最新稳定版。删除已过时实践(如旧版本兼容层、废弃 API 建议)。写作原则:只保留可验证的硬约束与决策性指引,删除描述性废话。

## 安装模型(README 重写核心)

仓库保持平台中立的 `skills/` + `rules/` + `AGENTS.md`;各 CLI 通过复制/软链接接入:

| CLI | Skills 接入 | 规范接入 |
|-----|------------|---------|
| Claude Code | 软链/复制到 `.claude/skills/` | `CLAUDE.md` 内 `@AGENTS.md` |
| Codex CLI | 软链/复制到 `.agents/skills/` | 原生读 AGENTS.md |
| OpenCode | 直接识别 `.claude/skills` / `.agents/skills` / `.opencode/skills` | 原生读 AGENTS.md |
| Grok Build | 项目 skills 目录或 `~/.grok/skills/` | 原生读 AGENTS.md(兼容 CLAUDE.md) |

## 不做的事

- 不保留任何多 Agent 编排文档或"模式选择"概念(单体/多 Agent 二分法随 agents/ 一起删除)。
- 不为单一平台写专属适配文件(.cursor/、.toml 等全部删除)。
- 不保留 rules/templates/ 下的工程模板(PR 清单、toolkit 等)。

## 风险与缓解

- **压缩丢失有价值条款**:压缩时以"可验证硬约束优先"为筛选标准,域内争议条款宁可保留。
- **版本信息时效**:重写时对不确定的版本号(如 Flutter)做联网核实,或采用"最新稳定版"表述避免硬编码过期版本。
- **git 历史**:全部通过 git 删除/移动,可随时回溯;旧内容不做归档目录(git 历史即归档)。
