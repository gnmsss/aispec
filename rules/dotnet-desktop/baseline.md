# .NET 桌面基线(必载)

适用:WPF / MAUI / WinForms 桌面应用。
冲突优先级:`rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. .NET 10 LTS 及以上(以 `TargetFramework` 为准);SDK-style `.csproj`;升级版本单独提交。
2. 依赖统一 NuGet;多项目用 `Directory.Build.props` + Central Package Management;禁止复制外部 DLL 进业务目录。
3. 全局启用 `<Nullable>enable</Nullable>`、`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`。
4. CI 集成 `dotnet list package --vulnerable`,高危漏洞阻断;商用禁止 GPL/AGPL;原生依赖(P/Invoke)特别审查;依赖更新单独提交。

## 架构(MUST)

1. WPF/MAUI 必须 MVVM(推荐 CommunityToolkit.Mvvm 源生成器);WinForms 推荐 MVP;同一项目禁止混用多套 MVVM/MVP 框架。
2. 分层单向:`View → ViewModel → Service → Repository`;启动入口(`App.xaml.cs`/`Program.cs`)只做 DI 配置与初始化。
3. View:禁止 Code-Behind 写业务逻辑(纯 UI 逻辑如动画、焦点除外);禁止直接实例化 ViewModel(DI 或 ViewModelLocator)。
4. ViewModel:禁止引用/操作 UI 控件(`Window`/`Control`/`MessageBox`);禁止持有 View 引用;对话框与文件选择必须经 `IDialogService` 等接口抽象。
5. Service:禁止依赖 ViewModel 或 UI 框架类型;View 禁止直接操作数据库/远程 API。
6. 多页面导航统一 `INavigationService`(DI 创建页面、强类型参数传递、离开时清理);禁止 ViewModel 直接 new Window/Page。
7. API 响应 DTO 与持久化实体禁止直接绑定 UI,必须在 Service/ViewModel 转换。

## 线程与异步(MUST)

1. UI 线程禁止耗时操作(I/O、网络、数据库、大量计算),一律异步;禁止 `.Result`/`.Wait()`/`GetAwaiter().GetResult()`。
2. 禁止 `async void`(事件处理器除外);禁止 fire-and-forget(`_ = DoAsync()`),必须处理异常。
3. 后台线程禁止直接改 UI 控件或 `ObservableCollection`,必须经 Dispatcher/`MainThread` 回 UI 线程。
4. 长耗时操作支持取消(`CancellationToken`)并有进度反馈。

## 依赖注入(MUST)

1. 统一 `Microsoft.Extensions.DependencyInjection`(Generic Host 模式优先);构造函数注入,禁止 Service Locator。
2. ViewModel 注册 Transient(或按需);禁止静态类/属性持有有状态组件;`HttpClient` 走 `IHttpClientFactory`。

## 数据与安全(MUST)

1. 敏感数据(密码、令牌、密钥)禁止明文存配置文件或本地库,必须用 DPAPI/系统凭据管理器/平台 SecureStorage。
2. 禁止硬编码 API Key、密钥、服务端凭据;禁止生产禁用 SSL 证书校验。
3. 用户数据存 `LocalApplicationData`(MAUI 用 `FileSystem.AppDataDirectory`),禁止写程序安装目录。
4. 本地数据库(SQLite/LiteDB)统一经 Repository 访问;Schema 变更走迁移管理。

## 代码质量(MUST)

1. 禁止提交 `Console.WriteLine`/`Debug.WriteLine`/`Debugger.Break()`/调试用 `MessageBox.Show` 到主分支;日志统一 `ILogger<T>` 结构化输出。
2. XAML 禁止硬编码颜色/字体/间距魔法值,必须定义为资源(ResourceDictionary/主题 token)。
3. 事件订阅必须配对取消(`-=` 或弱事件),对象销毁时清理;禁止循环字符串 `+=` 拼接(用 `StringBuilder`)。
4. 大图片必须设置解码尺寸(`DecodePixelWidth/Height`),禁止原始分辨率加载后缩放。

## 红线(MUST NOT)

1. 禁止 Code-Behind 业务逻辑;禁止 ViewModel 触碰 UI 类型。
2. 禁止 UI 线程阻塞与后台线程改 UI。
3. 禁止分发 Debug 版本到生产;发布必须代码签名。
4. 禁止明文存储敏感数据;禁止用户数据写安装目录。
5. 禁止未评审引入新 UI/MVVM/ORM 框架。
