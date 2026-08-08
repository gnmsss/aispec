# .NET 桌面场景实践(按需加载)

硬约束与红线见 `baseline.md`。按 profile 读取:WPF / MAUI / WinForms。

## 项目结构(WPF 为例,MAUI/WinForms 同构)

```text
MyApp/
├── src/
│   ├── MyApp.Desktop/            # 启动项目:App.xaml(.cs) DI 配置、Views/、Resources/(主题、样式 token)
│   ├── MyApp.ViewModels/         # ViewModel(可测试,不依赖 UI 框架):按功能模块分目录 + ViewModelBase
│   ├── MyApp.Services/           # 业务服务 + 接口抽象(IDialogService、INavigationService、IApiClient)
│   ├── MyApp.Data/               # 本地数据访问(SQLite Repository、迁移)+ 远程 API 客户端实现
│   └── MyApp.Shared/             # Options、常量、领域模型(无 UI 依赖)
└── tests/                        # ViewModel 与 Service 单元测试
```

1. MVVM 框架统一 CommunityToolkit.Mvvm:`[ObservableProperty]`、`[RelayCommand]` 源生成器,禁止手写样板 `INotifyPropertyChanged`。
2. MAUI:平台差异代码放 `Platforms/`,能力经 `#if` 最小化 + 接口抽象;页面导航用 Shell 路由。
3. WinForms:MVP 模式,Presenter 可测试,Form 仅做事件转发与显示。

## 启动与生命周期

1. Generic Host 模式:`Host.CreateApplicationBuilder` 配置 DI、日志、配置;`App` 启动时构建 Host,退出时 `StopAsync` 释放。
2. 全局异常处理启动即注册:`DispatcherUnhandledException` + `AppDomain.UnhandledException` + `TaskScheduler.UnobservedTaskException`,记录日志 + 用户友好提示,避免静默崩溃。
3. 单实例应用用 Mutex/命名管道检测,二次启动激活已有窗口。
4. 启动优化:主窗口尽快可见,重初始化异步执行,启动画面(Splash)覆盖初始化期。

## 数据访问

1. 本地数据库 SQLite:EF Core(`Microsoft.EntityFrameworkCore.Sqlite`)或 sqlite-net,统一选型;数据库文件在 `LocalApplicationData`;Schema 变更走 EF 迁移或版本化脚本。
2. 远程 API:`IHttpClientFactory` + 类型化客户端;超时与重试(Polly);Token 存系统凭据管理器,请求时读取。
3. 离线优先(如适用):本地缓存兜底,网络恢复后同步;同步冲突策略明确。

## 配置

1. `appsettings.json`(默认,随安装)+ 用户配置(写 `LocalApplicationData`,运行时可改);Options Pattern 强类型绑定。
2. 环境区分(开发/生产 API 地址)经构建配置或首次运行配置,不硬编码。

## UI 与主题

1. 颜色、字体、间距集中定义 ResourceDictionary(浅/深主题两套 token);控件样式统一资源字典分文件组织。
2. 长列表虚拟化(`VirtualizingStackPanel`/`CollectionView`);集合更新用 `ObservableCollection` 批量替换减少刷新。
3. 数据绑定错误当缺陷处理:开发期开启绑定跟踪(`PresentationTraceSources`),CI 构建无绑定警告。
4. 视觉与交互遵循 `rules/design.md`(无障碍:键盘导航、AutomationProperties、对比度)。
5. View 组件化:页面级 XAML 按语义区块拆为 UserControl(通用控件无业务语义放共享控件库,业务控件就近放功能目录);避免巨型 XAML 单文件,以「区块可独立预览、可复用」为拆分标准;ViewModel 随视图同步拆分。

## 自动更新

1. 更新方案统一选型:Windows 用 Velopack(Squirrel 后继)或 MSIX 自动更新;MAUI 移动端走商店更新。
2. 启动静默检测 + 手动检测入口;下载有进度;签名验证强制;失败可回退不破坏用户数据。
3. 更新渠道(stable/beta)与灰度按需配置;版本号 SemVer。

## 日志与诊断

1. `ILogger<T>` + Serilog File sink:日志写 `LocalApplicationData/logs`,滚动策略(大小/天数)可配;提供「打开日志目录」入口。
2. 崩溃转储与错误上报(用户同意后)带应用版本与系统信息;禁止记录敏感数据。

## 性能

1. 冷启动目标 ≤ 3s(可感知可用);内存关注:事件泄漏、图片缓存、长生命周期集合。
2. 大数据渲染分页/虚拟化;计算密集任务 `Task.Run` 后台执行;进度经 `IProgress<T>` 回报。
3. 诊断用 Visual Studio Profiler / dotnet-counters / PerfView。

## 测试与发布

1. ViewModel 与 Service 单元测试(xUnit + NSubstitute),对话框/导航经接口 Mock;Repository 用内存/临时 SQLite。
2. CI:build(0 warning)+ test + 打包;发布 Release 构建 + 代码签名(Authenticode/MSIX 签名)。
3. 安装包:MSIX 或 Inno Setup/Velopack 统一选型;卸载清理用户数据策略明确(默认保留,提供彻底清除选项)。
