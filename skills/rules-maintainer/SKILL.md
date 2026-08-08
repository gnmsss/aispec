---
name: rules-maintainer
description: AISpec 规范维护工具。新增、修改、校验本仓库 rules/ 与 skills/ 文件时使用。确保结构一致性、引用完整性与行数预算。
---

# 规范维护工具

维护本仓库规范体系的一致性。适用于:新增/修改域规则、新增/修改 skill、结构校验。

## 仓库结构契约

1. 每个技术域:`rules/<域>/baseline.md`(≤150 行,硬约束 + 红线)+ `rules/<域>/practices.md`(≤300 行,场景实践);对应 `skills/<域>/SKILL.md`(编码引导 + 审查清单 + 脚手架三节)。
2. 跨域单文件(≤250 行):`rules/database.md`、`security.md`、`collaboration.md`、`design.md`、`testing.md`、`operations.md`。
3. 入口:`AGENTS.md`(域索引 + 加载策略 + 冲突仲裁);`CLAUDE.md` 仅导入 AGENTS.md。
4. SKILL.md frontmatter 仅 `name`(== 目录名)+ `description`(含触发场景);禁止自定义字段与 `$skill` 引用语法。

## 维护流程

### 新增/修改规则

1. 定位归属:域内条款进对应域文件;跨域条款进 6 个跨域文件之一;判断进 baseline(可验证硬约束/红线)还是 practices(场景指引)。
2. 写作标准:只写可验证的硬约束与决策性指引;MUST/禁止措辞明确;不写背景废话;同主题条款合并不重复。
3. 修改后校验行数预算;超预算先删低价值条款再新增。
4. 新增技术域时同步:域规则 2 文件 + 域 skill + AGENTS.md 域索引表 + README 覆盖范围。

### 一致性校验清单

对变更文件逐项检查:

1. 文件内 `rules/...`、`skills/...` 引用路径全部真实存在
2. SKILL.md frontmatter 合规(name == 目录名,仅两个字段)
3. 行数在预算内(`wc -l`)
4. 无对已删除概念的引用:agents/、协作协议、execution-trace、task-router、$xxx skill 语法、rules/index.md、templates/
5. 冲突优先级声明与 AGENTS.md 一致(database > security > collaboration > 跨域 > 域内)
6. 版本信息未过时(框架版本表述与当前 stable 一致,拿不准就用「最新稳定版」)

### 输出

变更摘要 + 校验结果(逐项通过/修复情况);发现历史遗留问题时列出并给修复建议。
