# rules/ios/common/code-style.md

## Swift 代码风格（MUST）

1. 遵循 [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)。
2. 格式化：推荐使用 `SwiftFormat`，配置文件 `.swiftformat` 纳入版本控制。
3. 命名规范：
   - 类型/协议/枚举：`PascalCase`（`UserRepository`、`NetworkService`）
   - 函数/属性/变量：`camelCase`（`fetchUser(byId:)`、`userName`）
   - 常量（全局/静态）：`camelCase`（Swift 惯例，非 `SCREAMING_CASE`）
   - 枚举 case：`camelCase`（`case networkError`）
   - 布尔属性以 `is`/`has`/`should` 开头（`isLoading`、`hasPermission`）
4. 函数命名遵循"动词 + 宾语"或"介词短语"风格：
   - `fetchUser(byId:)` 而非 `getUserById()`
   - `remove(at:)` 而非 `removeAtIndex()`
5. 公开 API（`public` / `open`）必须有 `///` 文档注释。
6. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
7. 使用最严格的访问控制：默认 `private`，按需逐步放宽到 `internal` / `public`。

## Swift 惯用写法（MUST）

1. 优先使用 `struct` 而非 `class`（值类型优先）。
2. 优先使用 `let` 而非 `var`。
3. 优先使用 `guard` 提前返回，减少嵌套。
4. 优先使用 `enum` 建模有限状态，而非字符串或整数常量。
5. 使用 `@MainActor` 标注需要主线程执行的类型和方法。
6. 使用 Swift Concurrency（`async`/`await`）替代回调和 Combine（新代码）。

## SwiftLint 配置（MUST）

1. 必须启用以下规则（最低要求）：
   ```yaml
   # .swiftlint.yml
   opt_in_rules:
     - force_unwrapping
     - implicitly_unwrapped_optional
     - empty_count
     - closure_spacing
     - contains_over_filter_count
     - discouraged_optional_boolean
     - modifier_order
   disabled_rules: []
   force_cast: error
   force_try: error
   force_unwrapping: error
   ```
2. 禁止通过 `// swiftlint:disable` 绕过 error 级别规则，除非有充分理由并在 PR 中说明。

## 检查方式
- Swift：`swiftlint lint --strict` + `swiftformat --lint`
- 阻断级别：阻断合并
