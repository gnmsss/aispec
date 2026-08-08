# iOS 基线(必载)

适用:iOS 原生应用(Swift,SwiftUI 为主,存量 UIKit 项目兼容)。
冲突优先级:`rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Swift 6.x 最新 stable;新项目禁止新增 Objective-C 源文件;启用严格并发检查(`SWIFT_STRICT_CONCURRENCY = complete`)。
2. Xcode 与 Apple 最新稳定版同步,团队统一版本;最低部署目标按用户分布,推荐 iOS 17+(可用 `@Observable` 宏)。
3. 依赖首选 Swift Package Manager;版本锁定(`.upToNextMinor`),禁止宽泛范围;新依赖评审(维护状态、许可证、安全);禁止 SPM+CocoaPods+Carthage 三者并存。
4. Build Settings 用 `.xcconfig` 管理,禁止 GUI 散改;必须支持命令行构建(xcodebuild/Fastlane);`.pbxproj` 冲突频繁时用 XcodeGen/Tuist 生成。
5. SwiftLint 配置入库,CI `swiftlint lint --strict` 阻断;推荐 SwiftFormat 自动格式化。

## 分层架构(MUST)

1. 三层单向:`View(SwiftUI/ViewModel) → Domain(用例、Repository 协议,纯 Swift) → Data(Repository 实现、网络、存储)`;Domain 不依赖 UIKit/SwiftUI。
2. View/ViewController 禁止直接访问网络层/数据库;仅经 ViewModel 取数;禁止模块循环依赖;禁止 NotificationCenter 替代标准数据流。
3. DI:初始化器注入(手动构造/Factory/swift-dependencies);协议与实现分离便于 Mock;禁止 Singleton 管理可变状态。
4. ViewModel:iOS 17+ 用 `@Observable` 宏(存量用 `ObservableObject`);状态用不可变 struct;`@MainActor` 标注保证主线程更新;禁止持有 View/UIViewController 引用。
5. 导航:SwiftUI 用 `NavigationStack` + 类型安全路由,集中定义;UIKit 用 Coordinator;Deep Link 统一路由分发。
6. 异步统一 Swift Concurrency(async/await + actor),新代码禁止回调嵌套与手写 GCD 线程管理。

## Swift 代码(MUST)

1. 禁止 `!` 强制解包(用 `guard let`/`if let`/`??`);禁止 `try!`(用 do-catch/`try?`);禁止 `as!`(用 `as?` + guard);禁止隐式解包可选(`String!`,`@IBOutlet` 除外)。
2. 禁止 `Any`/`AnyObject` 替代具体类型(泛型或协议约束);禁止 Method Swizzling(评审批准除外)。
3. 主线程禁止:网络请求、数据库读写、文件 I/O;禁止后台线程 `DispatchQueue.main.sync`(死锁)。

## 安全(MUST)

1. 禁止硬编码密钥/Token/密码(xcconfig + CI 注入);禁止 `NSAllowsArbitraryLoads = YES`(ATS 全局豁免);禁止关闭 SSL 证书校验。
2. 密码/Token 存 Keychain,禁止 UserDefaults;日志禁止输出敏感信息。
3. 签名证书/私钥禁止入库;禁止私有 API(审核拒绝);生产构建禁止保留测试后门。

## 存储与 UI 资源(MUST)

1. UserDefaults 只存轻量偏好;缓存放 Caches 目录(禁止 Documents);禁止硬编码文件路径(用 FileManager API)。
2. 字符串必须 String Catalog(`.xcstrings`)本地化,禁止硬编码;颜色用 Asset Catalog Color Set;单位用 `pt`。
3. 布局必须尊重 Safe Area;视觉与无障碍遵循 `rules/design.md`(VoiceOver 标签、Dynamic Type、对比度)。

## 红线(MUST NOT)

1. 禁止 View 直接数据访问、ViewModel 持有 UI 引用。
2. 禁止强制解包三件套(`!`/`try!`/`as!`)进入主分支。
3. 禁止发布未测试构建;禁止跳过签名分发 IPA。
4. 禁止 Core Data/SwiftData 模型变更不做版本迁移。
