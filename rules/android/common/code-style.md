# rules/android/common/code-style.md

## Kotlin 代码风格（MUST）

1. 格式化：必须使用 `ktlint`，配置文件 `.editorconfig` 纳入版本控制。
2. 命名规范：
   - 类/接口/枚举/注解：`PascalCase`（`UserRepository`、`NetworkModule`）
   - 函数/属性/变量：`camelCase`（`getUserById`、`userName`）
   - 常量（`const val` / `companion object`）：`SCREAMING_SNAKE_CASE`（`MAX_RETRY_COUNT`）
   - 包名：全小写，不使用下划线（`com.example.feature.auth`）
   - 资源文件：`snake_case`（`ic_user_avatar.xml`、`activity_main.xml`）
3. Composable 函数命名使用 `PascalCase`（`UserProfileCard`），非 Composable 函数使用 `camelCase`。
4. 公开 API（`public` / `internal` 类与函数）必须有 KDoc 文档注释。
5. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
6. 优先使用 Kotlin 惯用写法：
   - 优先 `data class` 而非手动实现 `equals`/`hashCode`
   - 优先 `sealed class`/`sealed interface` 建模有限状态
   - 优先 `when` 表达式而非 `if-else` 链
   - 优先 `val` 而非 `var`，优先不可变集合

## 静态分析规则（MUST）

1. detekt 必须启用以下规则集：`complexity`、`style`、`potential-bugs`、`performance`。
2. detekt 自定义阈值建议：
   ```yaml
   complexity:
     LongMethod:
       threshold: 30
     ComplexMethod:
       threshold: 15
     LargeClass:
       threshold: 300
   ```
3. 禁止通过 `@Suppress` 绕过 detekt 规则，除非有充分理由并在 PR 中说明。

## 检查方式
- Kotlin：`./gradlew ktlintCheck` + `./gradlew detekt`
- Android Lint：`./gradlew lint`
- 阻断级别：阻断合并
