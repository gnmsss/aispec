# 贡献指南

感谢你对 AISpec 的贡献!以下是参与本项目的指南。

## 项目结构

```
aispec/
├── AGENTS.md   ← 唯一入口(域索引 + 加载策略 + 冲突仲裁)
├── rules/      ← 规则真源(每域 baseline + practices,跨域单文件)
└── skills/     ← 任务入口(标准 SKILL.md)
```

## 规则贡献(rules/)

- **归属**:域内条款进 `rules/<域>/`,跨域条款进 6 个跨域单文件之一;可验证硬约束/红线进 `baseline.md`,场景指引进 `practices.md`。
- **格式**:关键条款标注 `MUST`(阻断级)或 `SHOULD`(建议级);只写可验证约束与决策性指引,不写描述性废话。
- **行数预算**:域 baseline ≤ 150 行、practices ≤ 300 行、跨域单文件 ≤ 250 行;超预算先精简再新增。
- **校验**:变更后使用 `rules-maintainer` skill 执行一致性校验(引用路径、frontmatter、行数、冲突优先级)。

## Skill 贡献(skills/)

- 每个 skill 一个目录,`SKILL.md` frontmatter 仅 `name`(与目录名一致)+ `description`(含触发场景)。
- 域 skill 固定三节:编码引导、代码审查清单、项目脚手架。
- 规则引用直接写 `rules/...` 相对路径,禁止自造引用语法。

## 新增技术域

同步完成四件事:`rules/<域>/{baseline,practices}.md`、`skills/<域>/SKILL.md`、`AGENTS.md` 域索引表、`README.md` 覆盖范围。

## 提交流程

1. Fork 本仓库,创建分支 `feature/add-{domain}-{feature}`。
2. 编写内容并用 `rules-maintainer` skill 校验。
3. 提交 PR,描述变更内容与原因。

## 提交消息规范

```
{type}: {description}

type: feat(新增规则/skill) | fix(修复内容错误) | refactor(结构调整) | docs | chore
```

## 审查标准

- 新增规则有明确依据(业界最佳实践、安全标准、团队约定)。
- 规则间无冲突(参考 AGENTS.md 冲突仲裁);MUST 级规则可验证。
- 版本相关表述与当前 stable 一致,不确定时用「最新稳定版」。
- 所有文案使用中文。

## 问题反馈

创建 Issue 并标注:`bug`(规则错误)/ `enhancement`(改进建议)/ `new-domain`(新增技术域)。
