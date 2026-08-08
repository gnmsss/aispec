---
name: android
description: Android 工程规范。编写或修改 Android 原生代码(Kotlin、Jetpack Compose、Room、Retrofit、Hilt)、审查 Android 代码变更、或初始化 Android 项目时使用。
---

# Android 规范

## 编码引导

1. 任何 Android 编码任务,先读 `rules/android/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/android/practices.md` 对应章节:项目结构、网络与数据、Compose UI、错误处理与可观测性、配置与构建、测试发布。
3. 跨域联动:
   - API 契约、联调 → `rules/collaboration.md`
   - 视觉、交互、无障碍 → `rules/design.md`
   - 密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:UI 未直接访问 DAO/API;Domain 纯 Kotlin;模块依赖单向
2. ViewModel:StateFlow 暴露状态;一次性事件走 SharedFlow/Channel;无 Context/View 引用
3. Kotlin:无 !!;无 GlobalScope;协程结构化并发;主线程无 I/O
4. Compose:状态提升;重组范围合理;列表有稳定 key;collectAsStateWithLifecycle
5. 安全:无硬编码密钥;HTTPS + 证书校验;敏感数据加密存储;日志无敏感信息
6. 资源:字符串走 strings.xml;颜色走 Theme;尺寸 dp/sp;长列表 Lazy 组件
7. Room:迁移版本化,无 fallbackToDestructiveMigration(生产),未修改历史迁移
8. 构建:Version Catalog;无动态版本号;ktlint/detekt/lint 通过
9. 发布:R8 混淆;签名密钥未入库;versionCode 递增
10. 测试:ViewModel/用例有单元测试;缺陷修复有回归测试

## 项目脚手架

初始化新 Android 项目时:

1. 确认(未指定必须先询问):minSdk、是否多模块、是否需要本地数据库/离线支持。
2. Kotlin 2.x + Compose + Gradle Kotlin DSL + Version Catalog;配置 ktlint + detekt + Hilt。
3. 按 `rules/android/practices.md`「项目结构」建 `app/core/feature` 模块(或单模块同构包结构)。
4. 落地骨架:Retrofit/OkHttp 封装(拦截器 + 错误映射)、Room + 迁移基建、designsystem 主题 token、导航宿主、Timber 日志、崩溃上报、BuildConfig 环境注入。
5. 首个 feature 按 `ui/domain/data` 打样,配 ViewModel 单元测试(Turbine + runTest)。
