# .NET 服务端基线(必载)

适用:所有 ASP.NET Core 服务端代码(Web API、gRPC、Worker Service、后台任务)。
冲突优先级:`rules/database.md` > `rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. .NET 10 LTS 及以上,以 `TargetFramework` 为准;升级 .NET 版本单独提交;必须使用 SDK-style `.csproj`。
2. 依赖统一走 NuGet;启用 `RestorePackagesWithLockFile`,`packages.lock.json` 入库;多项目用 `Directory.Build.props` + Central Package Management(`Directory.Packages.props`)统一版本。
3. 全局启用 `<Nullable>enable</Nullable>`、`<ImplicitUsings>enable</ImplicitUsings>`;新项目启用 `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`。
4. CI 集成 `dotnet list package --vulnerable`,高危漏洞(CVSS ≥ 7.0)阻断合并;商用项目禁止 GPL/AGPL 依赖;依赖更新与业务代码分开提交。
5. 合并门禁:`dotnet format --verify-no-changes`、`dotnet build`(0 warning)、`dotnet test` 全部通过。

## 分层架构(MUST)

1. 解决方案分层:`Api`(协议适配)→ `Application`(用例编排、事务边界)→ `Domain`(实体、领域规则、业务异常、Repository 接口)→ `Infrastructure`(数据访问、外部服务实现);依赖单向,禁止循环。
2. `Program.cs` 仅做服务注册、中间件管道配置、生命周期管理,不承载业务逻辑。
3. Controller/Endpoint 禁止直接操作 `DbContext`,数据访问必须经 Application → Repository。
4. 协议层 DTO 不下沉到 Application/Domain;持久化实体不直接透传到对外 API;层间转换显式实现。
5. 共享类库(`*.Shared`)只放技术组件(Options、常量),禁止放业务实体和业务异常。

## 异步(MUST)

1. 异步贯穿全链路 async/await;禁止 `.Result`、`.Wait()`、`.GetAwaiter().GetResult()` 阻塞调用。
2. 禁止 `async void`(事件处理器除外);I/O 方法必须接收 `CancellationToken` 并透传。
3. `HttpClient` 必须通过 `IHttpClientFactory` 管理,禁止每次 `new HttpClient()`。

## 错误处理(MUST)

1. 业务异常与系统异常分离:业务异常继承统一 `BusinessException` 基类,携带结构化错误码(如 `USER_NOT_FOUND`,禁止纯数字);错误码常量集中定义在 Domain 层。
2. 统一异常处理:使用 `IExceptionHandler`(.NET 8+)或全局中间件,业务异常映射业务错误码,系统异常记录完整日志后返回固定文案;禁止在多个 Controller 分散映射。
3. 禁止空 `catch` 吞异常(显式注释说明的场景除外);系统异常(驱动异常、堆栈)禁止原样返回调用方。
4. 参数校验用 FluentValidation 或 DataAnnotations 统一在入口层执行,校验失败返回 400 + 字段级 details。

## API 响应(MUST)

1. 统一响应结构:`code`、`message`、`data`、`request_id`、`timestamp`;`request_id` 由中间件注入。
2. 语义化状态码:2xx/4xx/5xx;禁止失败请求统一返回 200;鉴权错误返回 401/403。
3. 对外时间统一 UTC;HTTP API 版本化(`/api/v1`),破坏性变更升版本。
4. 列表接口必须分页并限制 `pageSize` 上限;写接口必须定义幂等策略。

## 配置(MUST)

1. `appsettings.json` + `appsettings.{Environment}.json`;环境经 `ASPNETCORE_ENVIRONMENT` 显式指定并白名单校验;启动日志输出当前环境。
2. 统一 Options Pattern:每个配置节有强类型类,`AddOptions<T>().BindConfiguration().ValidateDataAnnotations().ValidateOnStart()` 注册,启动即校验;禁止业务代码直接读 `IConfiguration` 键值。
3. 密钥来自安全存储(Key Vault/Secrets Manager/配置中心),禁止入库;`appsettings.*` 仅存非敏感默认值。
4. 数据库必须显式声明 `Database:Type`(仅 `mysql` → Pomelo 或 `postgresql` → Npgsql);未指定时先询问。
5. CORS 白名单走配置、支持多域名;`AllowCredentials` 时禁止 `AllowAnyOrigin`。

## 日志(MUST)

1. 统一结构化日志(`ILogger<T>` + Serilog 或内置 Provider),使用消息模板与命名参数,禁止字符串插值拼日志。
2. 禁止 `Console.WriteLine` 调试残留提交主分支;每条请求链路记录 `request_id`/`trace_id`。
3. 禁止输出密码、令牌、证件号等敏感信息;生产环境禁止启用 `EnableSensitiveDataLogging`。

## 红线(MUST NOT)

1. 禁止 Controller/Endpoint 直接操作 `DbContext`。
2. 禁止静态类/静态属性持有 `DbContext`、Redis 连接等有状态组件;禁止 Service Locator(`IServiceProvider.GetService`)替代构造函数注入;禁止手动 `new DbContext()`。
3. 禁止无上限缓存(`IMemoryCache` 必须设 `SizeLimit`)、无上限队列、无超时外部调用。
4. 禁止 `IQueryable` 先 `ToList()` 再内存过滤,过滤必须发生在数据库端。
5. 禁止未评审的破坏性接口变更和数据库结构变更;结构变更遵循 `rules/database.md`,禁止手改生产库。
6. 禁止硬编码外部依赖地址与凭据;禁止 CORS 域名写死在代码。
