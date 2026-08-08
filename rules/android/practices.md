# Android 场景实践(按需加载)

硬约束与红线见 `baseline.md`。默认 Compose;XML Views 项目按等价模式执行。

## 项目结构(多模块 feature-first)

```text
app/                          # 装配:Application、MainActivity、导航宿主、Hilt 入口
core/
├── network/                  # Retrofit/OkHttp 封装、拦截器、错误映射
├── database/                 # Room:AppDatabase、迁移
├── datastore/                # Preferences DataStore 封装
├── designsystem/             # 主题、Material3 token、通用组件
├── common/                   # 工具、扩展、Result 封装
feature/<feature>/
├── ui/                       # Screen Composable + ViewModel + UiState
├── domain/                   # 用例、Repository 接口(纯 Kotlin)
└── data/                     # Repository 实现、DataSource、DTO
gradle/libs.versions.toml     # Version Catalog
build-logic/                  # Convention Plugins
```

1. feature 间禁止互相依赖,共享内容下沉 core;模块依赖单向:app → feature → core。
2. 小型项目可单模块但保持同样包结构,增长后再拆模块。

## 网络与数据

1. 网络:Retrofit + OkHttp + kotlinx.serialization;拦截器统一鉴权注入、刷新令牌、日志(脱敏);超时显式配置;错误映射为统一 sealed `Result`/`Failure` 类型。
2. Room:实体、DAO、Database 在 core/database;迁移版本化;查询返回 `Flow` 支持响应式;DAO 方法挂起函数。
3. Preferences DataStore 替代 SharedPreferences(非敏感);敏感数据 EncryptedSharedPreferences/Keystore。
4. Repository:单一数据源原则(数据库为 UI 数据源,网络负责同步);离线优先场景配同步策略与冲突处理。
5. 分页用 Paging 3;后台任务用 WorkManager(约束、重试、唯一工作)。

## Compose UI

1. Screen 模式:`XxxScreen(viewModel)` 收集 `StateFlow` → `XxxContent(state, callbacks)` 纯展示可预览;状态提升,Composable 无副作用逻辑。
2. `UiState` 用 sealed/data class 表达 loading/error/data;事件回调向上传,禁止 Composable 直接调 ViewModel 外的业务。
3. 性能:列表 item 提供稳定 key;重组范围最小化(状态下沉、`derivedStateOf`、lambda 稳定性);大列表图片用 Coil 并指定尺寸。
4. 副作用 API 正确使用:`LaunchedEffect`(键控)、`DisposableEffect`(清理)、`collectAsStateWithLifecycle()`(生命周期感知收集)。
5. 预览:关键 Composable 提供 `@Preview`(浅/深主题、字体缩放)。

## 错误处理与可观测性

1. 全局兜底:`Thread.setDefaultUncaughtExceptionHandler` + 协程 `CoroutineExceptionHandler`;崩溃上报接入 Crashlytics/Sentry。
2. 错误分级:业务错误(UI 可见提示,统一组件)/系统错误(上报 + 通用提示);网络错误统一映射(超时/无网/服务端错误)。
3. 日志统一封装(Timber 或等效):Debug 全量,Release 仅 WARN+ 且接入上报;禁止 `Log.d` 散写。
4. 性能监控:启动耗时、ANR、慢帧纳入监控(Firebase Performance/自建);基准测试用 Macrobenchmark(关键路径)。

## 配置与构建

1. BuildConfig 注入环境差异(API 地址等),值来自 `local.properties`/CI 环境变量,不入库;多环境用 productFlavors(dev/prod)。
2. R8 混淆规则维护入库;资源收缩开启;Baseline Profile 生成提升启动性能。
3. 版本号:`versionName` SemVer + `versionCode` 单调递增,CI 自动注入。

## 测试与发布

1. 单元测试:ViewModel(Turbine 测 Flow)+ 用例 + Repository(mock DataSource);协程用 `runTest` + 注入 TestDispatcher。
2. UI 测试:Compose Testing(`createComposeRule`)覆盖关键交互;截图测试(Roborazzi/Paparazzi)做视觉回归(可选)。
3. CI:ktlintCheck + detekt + lint + testDebugUnitTest + assembleRelease;签名走 CI Secret。
4. 发布:AAB 上传 Play(分阶段放量),崩溃率阈值回滚;国内渠道按需多渠道打包;Release Notes 与版本 tag 同步。
