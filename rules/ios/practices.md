# iOS 场景实践(按需加载)

硬约束与红线见 `baseline.md`。默认 SwiftUI + Swift Concurrency;UIKit 项目按等价模式执行。

## 项目结构(feature-first)

```text
MyApp/
├── App/                        # 入口:MyApp.swift、AppDelegate(如需)、根路由、DI 组装
├── Core/
│   ├── Network/                # URLSession/API Client 封装、拦截、错误映射
│   ├── Storage/                # SwiftData/Core Data 栈、Keychain 封装、文件管理
│   ├── DesignSystem/           # 颜色/字体/间距 token、通用组件
│   └── Common/                 # 扩展、工具、AppError 定义
├── Features/<Feature>/
│   ├── Views/                  # SwiftUI View
│   ├── ViewModels/             # @Observable ViewModel
│   ├── Domain/                 # 模型、用例、Repository 协议
│   └── Data/                   # Repository 实现、DTO、DataSource
├── Resources/                  # Assets、Localizable.xcstrings
└── Tests/ / UITests/
```

1. Feature 间禁止互相引用内部实现,共享内容下沉 Core;大型项目按 SPM 本地 Package 拆模块。
2. 项目生成推荐 Tuist/XcodeGen,减少 `.pbxproj` 冲突。

## 网络与数据

1. API Client 基于 URLSession + async/await 统一封装:鉴权注入、刷新令牌、超时、重试、日志脱敏;错误映射为统一 `AppError`(enum,含网络/业务/系统分类)。
2. DTO 用 `Codable` struct,与领域模型分离;日期与金额解析策略统一(ISO 8601、Decimal)。
3. 本地持久化:新项目优先 SwiftData(iOS 17+),存量 Core Data;模型变更做版本迁移;查询在后台上下文,主线程仅取展示数据。
4. Keychain 封装统一接口(读写删 + 生物识别保护可选);UserDefaults 只存轻量偏好。
5. Repository:协议在 Domain,实现组合远端 + 本地;缓存失效策略明确。

## SwiftUI

1. View 模式:`XxxScreen`(接 ViewModel,处理状态分发)→ 子 View 纯展示可预览;状态最小化下沉,`@State` 仅限视图私有状态。
2. ViewModel `@Observable` + `@MainActor`;异步加载用 `.task` 修饰符(自动取消);三态(loading/error/data)统一组件呈现。
3. 列表用 `List`/`LazyVStack`,item 提供稳定 `id`;图片异步加载(AsyncImage/Nuke)并限制尺寸。
4. 预览:关键 View 提供 `#Preview`(浅/深模式、Dynamic Type 大字号)。
5. 性能:避免大 View body 重算(拆分子 View、`EquatableView` 按需);Instruments(SwiftUI 模板)排查重绘。

## 并发

1. Swift Concurrency 全面使用:`async/await`、`actor` 保护共享可变状态、`TaskGroup` 并行编排;遵守 Sendable 检查。
2. 任务生命周期:View 层用 `.task`;ViewModel 内部长任务持有 `Task` 引用并在取消时清理。
3. 阻塞操作(压缩、大文件)放 `Task.detached` 或专用 actor,不占用主 actor。

## 错误处理与可观测性

1. `AppError` 分级:业务错误(用户可见提示,统一 Alert/Toast 组件)/系统错误(上报 + 通用提示);本地化错误文案。
2. 崩溃上报接入 Crashlytics/Sentry;关键流程埋点(启动、登录、支付)带版本与设备信息,脱敏。
3. 日志统一 `os.Logger`(subsystem/category 分类);Release 不输出 debug 级;MetricKit 收集启动/卡顿/耗电指标。

## 配置与构建

1. 多环境:xcconfig(Dev/Staging/Prod)+ scheme 区分;API 地址、开关走 xcconfig 注入 Info.plist 读取;密钥经 CI 注入不入库。
2. 版本号:MARKETING_VERSION SemVer + CURRENT_PROJECT_VERSION 递增,CI 自动注入。
3. 构建自动化:Fastlane(build/test/beta/release lane);证书管理用 match 或 App Store Connect API Key。

## 测试与发布

1. 单元测试:Swift Testing(`@Test`,新项目优先)或 XCTest;ViewModel 与用例覆盖正常/边界/异常;Repository mock DataSource;异步测试用 confirmation/async 断言。
2. UI 测试覆盖核心链路(登录、主流程);快照测试(可选)做视觉回归。
3. CI:SwiftLint strict + 单元测试 + 构建;PR 阻断。
4. 发布:TestFlight 内测 → 分阶段发布(Phased Release);崩溃率监控回滚阈值;App Store 审核清单(权限文案、隐私清单 PrivacyInfo.xcprivacy、第三方 SDK 声明)过检。
