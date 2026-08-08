# Android 基线(必载)

适用:Android 原生应用(Kotlin,Compose 为主,存量 XML Views 项目兼容)。
冲突优先级:`rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Kotlin 2.x 最新 stable(以 `gradle/libs.versions.toml` 为准);新项目禁止新增 Java 源文件;新 UI 默认 Jetpack Compose。
2. `compileSdk` 跟进最新稳定 API Level;`targetSdk` 满足 Google Play 最新上架要求(2026:API 36+);`minSdk` 按用户分布,推荐 ≥ 26。
3. 构建:Gradle Kotlin DSL(`build.gradle.kts`)+ Version Catalog 强制;多模块用 Convention Plugins;提交前 `./gradlew build` 无错误。
4. 依赖:禁止 `+` 动态版本;新依赖评审(维护状态、许可证、安全);漏洞扫描(dependencyCheck/Snyk)高危阻断;app 模块用 `implementation` 隔离,禁止随意 `api` 传递。
5. 静态分析:ktlint + detekt 配置入库,CI 阻断;Android Lint Error 级阻断。

## 分层架构(MUST)

1. 三层单向:`UI(Compose/ViewModel) → Domain(用例、Repository 接口,纯 Kotlin) → Data(Repository 实现、网络、数据库)`;Domain 不依赖 Android Framework。
2. UI 层禁止直接访问 DAO/API Service;仅经 ViewModel 取数;禁止模块循环依赖;禁止 EventBus/LocalBroadcastManager 通信。
3. DI 统一 Hilt:构造函数注入(字段注入仅限 Android 组件);Module 按职责拆分(Network/Database/Repository)。
4. ViewModel:UI 状态用 `StateFlow<UiState>` 暴露(新项目禁止 LiveData);一次性事件用 `SharedFlow`/`Channel`;禁止持有 `Context`/`View`/`Activity`(需要时 `@ApplicationContext`)。
5. 导航:Navigation Compose,路由集中定义禁止散写字符串;跨模块导航经接口/DeepLink 解耦。

## Kotlin 代码(MUST)

1. 禁止 `!!`(用 `?.`、`requireNotNull()`、`checkNotNull()`);优先 `val`;禁止 `Any` 替代具体类型;`@Suppress` 必须注释原因。
2. 协程结构化并发:禁止 `GlobalScope.launch`,用 `viewModelScope`/`lifecycleScope`;挂起函数主线程安全(内部切 `Dispatchers.IO`)。
3. 主线程禁止:网络请求、数据库读写、文件 I/O、`Thread.sleep()`。

## 安全(MUST)

1. 禁止硬编码密钥/Token/密码;禁止明文 HTTP(强制 HTTPS + 不关闭证书校验);敏感配置经 BuildConfig 注入且不入库。
2. 敏感信息存储:EncryptedSharedPreferences / Keystore,禁止裸 SharedPreferences 与外部存储(`/sdcard/`)。
3. 日志禁止输出敏感信息;Release 构建禁止 `debuggable = true`。

## UI 资源(MUST)

1. 字符串必须 `strings.xml`(支持国际化),禁止硬编码;颜色经 Theme/Material token 引用;尺寸用 `dp`/`sp` 禁止 `px`。
2. 长列表用 `LazyColumn`/`RecyclerView`,禁止 ScrollView 嵌套动态列表。
3. 视觉与无障碍遵循 `rules/design.md`(contentDescription、触控目标 ≥ 48dp、深色模式)。

## 红线(MUST NOT)

1. 禁止生产使用 `fallbackToDestructiveMigration()`;Room 迁移版本化入库,禁止修改历史迁移。
2. 禁止发布未混淆(R8)的 Release;禁止跳过签名;签名密钥禁止入库(CI Secret 管理)。
3. 禁止未跑 `testDebugUnitTest` 发布。
4. 禁止 UI 层直接数据访问、ViewModel 持有 UI 引用。
