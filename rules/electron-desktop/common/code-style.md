# rules/electron-desktop/common/code-style.md

## 主进程代码风格（MUST）

1. 格式化：必须使用 Prettier，配置文件 `.prettierrc` 纳入版本控制。
2. 命名规范：
   - 类/接口：`PascalCase`（`WindowManager`、`UpdateService`）
   - 函数/方法/变量：`camelCase`（`checkUpdate`、`userName`）
   - 常量：`UPPER_SNAKE_CASE`（`MAX_RETRY_COUNT`）
   - 文件/模块：`camelCase` 或 `kebab-case`（`windowManager.ts`、`update-service.ts`）
3. IPC channel 命名使用 `kebab-case` 加域前缀（如 `user:get-profile`、`file:read-content`）。
4. 公开模块导出必须有 JSDoc/TSDoc 注释。
5. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
6. 禁止使用 `any` 类型，必须 `"strict": true`。
7. 禁止在主进程代码中使用 `console.log` 调试输出，必须使用结构化日志库。

## preload 脚本代码风格（MUST）

1. preload 脚本必须使用 TypeScript 编写。
2. preload 仅暴露最小必要 API，禁止暴露完整 `ipcRenderer` 对象。
3. 暴露的 API 必须定义完整的 TypeScript 类型。
4. preload 脚本中禁止包含业务逻辑。

## 渲染进程代码风格（MUST）

1. 渲染进程代码风格遵循 `rules/frontend` 中的对应规范。
2. IPC 调用必须封装为独立的 API 层，禁止在组件中直接调用 `window.electronAPI`。
3. IPC 调用的参数和返回值必须定义 TypeScript 类型。

## 检查方式
- 主进程/preload：ESLint + Prettier
- 渲染进程：ESLint + Prettier
- 阻断级别：阻断合并
