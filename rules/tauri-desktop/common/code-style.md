# rules/tauri-desktop/common/code-style.md

## Rust 代码风格（MUST）

1. 格式化：必须使用 `rustfmt`，配置文件 `rustfmt.toml` 纳入版本控制。
2. 命名规范：
   - 类型/Trait：`PascalCase`（`AppState`、`UpdateManager`）
   - 函数/方法/变量：`snake_case`（`check_update`、`user_name`）
   - 常量：`SCREAMING_SNAKE_CASE`（`MAX_RETRY_COUNT`）
   - 模块/文件：`snake_case`（`auto_update.rs`）
3. Tauri Command 函数命名使用 `snake_case`，前端调用时自动转为 `camelCase`。
4. 公开 API（`pub fn`、`pub struct`）必须有 `///` 文档注释。
5. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
6. 禁止 `unwrap()` 出现在生产代码中，使用 `?` 操作符或 `expect("明确原因")`。
7. 禁止 `unsafe` 块，除非有充分理由并在 PR 中说明。

## 前端代码风格（MUST）

1. 必须使用 TypeScript（`strict: true`），禁止 `any` 类型。
2. 前端代码风格遵循 `rules/frontend` 中的对应规范。
3. Tauri IPC 调用必须封装为独立的 API 层，禁止在组件中直接调用 `invoke()`。
4. IPC 调用的参数和返回值必须定义 TypeScript 类型。

## 检查方式
- Rust：`cargo fmt --check` + `cargo clippy -- -D warnings`
- 前端：ESLint + Prettier
- 阻断级别：阻断合并
