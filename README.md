<div align="center">

# AISpec

**AI 工程规范体系 — 让 AI 编码助手输出一致、高质量、符合工程规范的代码**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

</div>

## 是什么

AI 编码助手能力强大,但缺乏统一工程规范约束,容易在不同会话、不同成员间产生风格不一致、最佳实践遗漏等问题。AISpec 提供一套结构化的**规则(rules)+ 技能(skills)**,让 AI 在编码、审查、脚手架、设计等环节始终遵循团队工程标准。

- **11 个技术域**:Go / .NET / Python / Node.js / Rust 服务端 · Vue3 + uni-app 前端 · Tauri / .NET 桌面 · Flutter / Android / iOS
- **7 类跨域规范**:数据库 · 安全 · 前后端协作 · 设计 · 测试 · 运维 · 项目文档
- **17 个 Skills**:每域一个三合一 skill(编码引导 + 审查清单 + 脚手架)+ 5 个横切 skill
- **标准格式**:AGENTS.md + Anthropic SKILL.md 格式,主流 AI CLI 开箱即用

## 体系结构

```
aispec/
├── AGENTS.md                 ← 唯一入口:域索引 + 加载策略 + 冲突仲裁
├── CLAUDE.md                 ← Claude Code 兼容(导入 AGENTS.md)
├── rules/                    ← 规则真源
│   ├── <域>/baseline.md      ← 必载:硬约束 + 红线
│   ├── <域>/practices.md     ← 按需:场景实践
│   └── database|security|collaboration|design|testing|operations|documentation.md
└── skills/                   ← 任务入口(标准 SKILL.md)
    ├── <域>/SKILL.md          ← 11 个域 skill + database
    └── spec-generator | security-auditor | design-guide | project-docs | rules-maintainer
```

## 快速开始

```bash
# 克隆到项目中(或作为 submodule)
git clone https://github.com/gnmsss/aispec.git
```

### 按 CLI 接入

| CLI | Skills 接入 | 项目规范接入 |
|-----|------------|-------------|
| **Claude Code** | 复制/软链 `skills/*` 到 `.claude/skills/` | `CLAUDE.md` 中 `@AGENTS.md` 导入(本仓库已内置) |
| **OpenAI Codex** | 复制/软链 `skills/*` 到 `.agents/skills/`(或全局 `~/.agents/skills/`) | 原生读取 AGENTS.md |
| **OpenCode** | 直接识别 `.claude/skills` / `.agents/skills` / `.opencode/skills` | 原生读取 AGENTS.md |
| **Grok Build** | 项目 skills 目录或 `~/.grok/skills/` | 原生读取 AGENTS.md(兼容 CLAUDE.md) |

示例(Claude Code,项目内):

```bash
# Windows (PowerShell,以仓库置于项目根 aispec/ 为例)
New-Item -ItemType Junction -Path .claude\skills -Target aispec\skills
Copy-Item aispec\AGENTS.md .; Copy-Item aispec\CLAUDE.md .

# macOS / Linux
ln -s ../aispec/skills .claude/skills
cp aispec/AGENTS.md aispec/CLAUDE.md .
```

接入后直接对 AI 下达任务即可——skill 会按任务类型自动触发,并按 `AGENTS.md` 的加载策略读取对应规则;也可显式调用(如 Claude Code 中 `/go-server`)。

## 覆盖范围与版本基线

| 类别 | 技术栈(2026-08 基线) |
|------|---------------------|
| 服务端 | Go 1.25+ · .NET 10 LTS · Python 3.12+(uv/ruff/Pydantic v2) · Node.js 22+ LTS(ESM/pnpm) · Rust stable(axum/tokio) |
| 前端 | Vue 3.5+ + Vite + Element Plus · uni-app(H5 + 微信小程序) |
| 桌面 | Tauri 2.x(Rust) · WPF / MAUI / WinForms(.NET 10) |
| 移动 | Flutter stable · Kotlin 2.x + Compose · Swift 6 + SwiftUI |
| 数据库 | MySQL / PostgreSQL(Schema 初始化 + 迁移管理) |

## 冲突仲裁

`database.md` > `security.md` > `collaboration.md` > 其余跨域 > 域内规则;同级以「更严格且可验证」为准。详见 [AGENTS.md](AGENTS.md)。

## 贡献

参与贡献请阅读 [CONTRIBUTING.md](CONTRIBUTING.md);修改规范时使用 `rules-maintainer` skill 保证一致性。

## 许可证

[MIT License](LICENSE) © 2026 AI Engineering Standard
