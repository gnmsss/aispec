---
name: ios
description: iOS 工程规范。编写或修改 iOS 原生代码(Swift、SwiftUI、SwiftData、Swift Concurrency)、审查 iOS 代码变更、或初始化 iOS 项目时使用。
---

# iOS 规范

## 编码引导

1. 任何 iOS 编码任务,先读 `rules/ios/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/ios/practices.md` 对应章节:项目结构、网络与数据、SwiftUI、并发、错误处理与可观测性、配置与构建、测试发布。
3. 跨域联动:
   - API 契约、联调 → `rules/collaboration.md`
   - 视觉、交互、无障碍 → `rules/design.md`
   - 密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:View 未直接访问网络/数据库;Domain 纯 Swift;依赖单向
2. Swift:无 !/try!/as! 强制解包;无隐式解包可选(IBOutlet 除外);无 Any 滥用
3. 并发:Swift Concurrency 统一;ViewModel @MainActor;主线程无 I/O;共享状态有 actor 保护;Sendable 检查通过
4. ViewModel:@Observable(或 ObservableObject);状态不可变 struct;无 UI 引用
5. 安全:无硬编码密钥;无 ATS 全局豁免;Token 在 Keychain;日志无敏感信息
6. 资源:字符串走 String Catalog;颜色走 Asset Catalog;Safe Area 尊重
7. 存储:缓存在 Caches;UserDefaults 只存偏好;模型变更有版本迁移
8. 导航:类型安全路由集中定义;无硬编码路由字符串
9. 构建:xcconfig 管理配置;SwiftLint strict 通过;证书未入库
10. 测试:ViewModel/用例有单元测试;缺陷修复有回归测试

## 项目脚手架

初始化新 iOS 项目时:

1. 确认(未指定必须先询问):最低部署目标(推荐 iOS 17+)、是否需要本地持久化(SwiftData)、是否用 Tuist/XcodeGen。
2. Swift 6 严格并发 + SwiftUI;配置 SwiftLint + SwiftFormat + xcconfig 多环境(Dev/Staging/Prod)。
3. 按 `rules/ios/practices.md`「项目结构」建 `App/Core/Features/Resources` 结构。
4. 落地骨架:API Client 封装(async/await + 错误映射)、AppError 体系、Keychain 封装、DesignSystem token、统一路由、os.Logger 日志、崩溃上报。
5. 首个 Feature 按 `Views/ViewModels/Domain/Data` 打样,配 Swift Testing 单元测试;Fastlane 配 build/test lane。
