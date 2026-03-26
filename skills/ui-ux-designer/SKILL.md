---
name: ui-ux-designer
description: UI/UX 设计师技能。当需要创建或更新界面设计方案（信息架构、用户流程、视觉设计、设计系统、设计 Token）时触发。通过四阶段引导（设计调研→交互设计→视觉设计→设计验证）产出完整的设计方案。依赖 Pencil MCP 创建可视化原型。
---

# UI/UX 设计师

通过四阶段引导，从设计调研到视觉设计到设计验证，产出完整的 UI/UX 设计方案。

## 何时使用
1. 新产品/新功能需要界面设计。
2. 用户要求设计页面、组件、设计系统。
3. 需要定义设计 Token（颜色/字体/间距/圆角/阴影）。
4. 由 Coordinator 在 Phase 0.5 调度时。

## 外部 MCP 依赖
- **Pencil MCP**（必需）：创建和编辑 `.pen` 设计文件。
- **搜索 MCP**（可选）：竞品设计调研。
- **Playwright MCP**（可选）：竞品界面截图和体验分析。
- MCP 不可用时按 `agents/adapters/mcp-tools.md` 的引导协议执行。

## 执行原则
1. 按四阶段逐步执行，每个阶段产出明确的中间产物。
2. 以 `references/design-scenario-map.md` 为阶段路由表。
3. 以 `rules/design/` 下的规范文件为设计约束。
4. 使用 Pencil MCP 时必须先调用 `get_guidelines`（topic 按场景选择：`web-app` / `mobile-app` / `design-system`）。
5. 设计 Token 必须导出为 CSS 变量格式，便于前端直接使用。
6. Pencil MCP 不可用时退化为文本化设计方案 + CSS 变量定义。

## 标准工作流（必须执行）

### Phase 1：设计调研
- 参考 `agents/design/phases/01-research.md` 中的引导问题。
- 分析竞品界面设计（风格/布局/字体/配色/组件/交互）。
- 确定设计风格方向和品牌色。
- 产出：设计调研报告。

### Phase 2：交互设计（UX）
- 参考 `agents/design/phases/02-ux.md` 中的引导问题。
- 设计信息架构图（页面层级、导航结构）。
- 设计用户操作流程。
- 定义完整页面清单和页面间跳转关系。
- 定义全局交互规则（加载/空状态/错误/成功反馈）。
- 确定响应式策略。
- 产出：信息架构图、用户流程图、页面清单。

### Phase 3：视觉设计（UI）
- 参考 `agents/design/phases/03-ui.md` 中的引导问题。
- 使用 Pencil MCP 的 `set_variables` 定义设计 Token。
- 使用 `insert_design_nodes` 创建组件和页面。
- 设计 Token 包括：色彩体系、字体体系、间距体系、圆角、阴影、动效时长。
- 导出 CSS 变量格式的设计 Token。
- 产出：设计 Token、组件库、所有页面设计（.pen 文件）。

### Phase 4：设计验证
- 参考 `agents/design/phases/04-review.md` 中的引导问题。
- 使用 `get_screenshot` 验证设计效果。
- 使用 `snapshot_layout` 检查布局问题。
- 检查设计完整性（所有页面是否覆盖）。
- 检查设计一致性（Token 使用是否统一）。
- 检查无障碍合规性（对比度、字号、可操作区域）。
- 产出：设计审查报告、设计还原度评分（A/B/C/D）、设计交付文档。
- 按 `agents/protocols/execution-trace.md` 格式，在输出末尾附执行追溯（skill 名称、任务类型、加载规则清单、跨域联动）。

## 资源
1. 阶段路由表：`references/design-scenario-map.md`
2. 引导问题模板：`agents/design/phases/`
3. 设计规范：`rules/design/` 下所有文件
4. MCP 工具方案：`agents/adapters/mcp-tools.md`
