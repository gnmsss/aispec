---
name: dotnet-desktop
description: .NET 桌面应用工程规范。编写或修改 WPF/MAUI/WinForms 桌面代码(MVVM、导航、本地数据、自动更新)、审查 .NET 桌面代码变更、或初始化 .NET 桌面项目时使用。
---

# .NET 桌面规范

## 编码引导

1. 任何 .NET 桌面编码任务,先读 `rules/dotnet-desktop/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/dotnet-desktop/practices.md` 对应章节:项目结构、启动与生命周期、数据访问、配置、UI 与主题、自动更新、日志、性能、测试发布。
3. 跨域联动:
   - 远程 API 契约 → `rules/collaboration.md`
   - 视觉、交互、无障碍 → `rules/design.md`
   - 密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:Code-Behind 无业务逻辑;ViewModel 无 UI 类型引用;对话框走 IDialogService;View 未直接操作数据/API
2. 异步:UI 线程无耗时操作;无 .Result/.Wait();无 async void(事件除外);无 fire-and-forget;后台线程未直接改 UI
3. MVVM:CommunityToolkit.Mvvm 源生成器;无手写样板;导航走 INavigationService 强类型参数
4. DI:构造函数注入;无 Service Locator;无静态持有有状态组件;HttpClient 走 Factory
5. 安全:敏感数据不明文存储;无硬编码密钥;用户数据在 LocalApplicationData;SSL 校验未禁用
6. 资源:XAML 无魔法值;事件订阅有取消;大图片有解码尺寸;长列表虚拟化
7. 日志:ILogger 结构化;无 Console/Debug.WriteLine 残留;无敏感信息
8. 构建:0 warning;Release + 签名发布;绑定无警告
9. 测试:ViewModel/Service 有单元测试;缺陷修复有回归测试

## 项目脚手架

初始化新 .NET 桌面项目时:

1. 确认(未指定必须先询问):UI 框架(WPF/MAUI/WinForms)、是否需要本地数据库、是否需要自动更新。
2. .NET 10 LTS;按 `rules/dotnet-desktop/practices.md`「项目结构」建 `Desktop/ViewModels/Services/Data/Shared` 多项目 + tests。
3. `.csproj` 启用 Nullable + TreatWarningsAsErrors;CommunityToolkit.Mvvm + Generic Host DI。
4. 落地骨架:全局异常处理三件套、Serilog 文件日志、IDialogService/INavigationService 抽象与实现、主题资源字典(token 化)、Options 配置。
5. 需要更新时配置 Velopack/MSIX;CI 配 build + test + 签名打包。
