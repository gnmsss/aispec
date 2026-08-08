---
name: dotnet-server
description: .NET 服务端工程规范。编写或修改 ASP.NET Core 代码(Web API、gRPC、Worker Service、后台任务)、审查 .NET 服务端代码变更、或初始化 .NET 服务端项目时使用。
---

# .NET 服务端规范

## 编码引导

1. 任何 .NET 服务端编码任务,先读 `rules/dotnet-server/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/dotnet-server/practices.md` 对应章节:解决方案结构、API 契约、组件初始化、EF Core 数据访问、缓存、后台任务、文件存储、可观测性、性能、测试;微服务项目额外读文末「微服务差异」。
3. 跨域联动:
   - API 契约变更 / 前后端联调 → `rules/collaboration.md`
   - 数据库 schema / 迁移 → `rules/database.md`(优先级最高)
   - 认证、密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:Api/Application/Domain/Infrastructure 依赖单向;Controller 未直接操作 DbContext
2. 异步:全链路 async/await,无 `.Result`/`.Wait()`,无 `async void`,`CancellationToken` 透传
3. 异常:无空 catch;业务异常带错误码走统一 `IExceptionHandler`;系统异常不外泄
4. 响应:统一包结构,语义化状态码,无 200 兜底
5. EF Core:只读查询有 `AsNoTracking`;LINQ 有投影;无 N+1;无 `ToList()` 后内存过滤
6. 事务:边界在 Application 层,事务内无远程调用
7. DI:构造函数注入,无 Service Locator,无静态持有有状态组件;HttpClient 走 `IHttpClientFactory`
8. 配置:Options Pattern + `ValidateOnStart`;无硬编码密钥/地址;`IMemoryCache` 有 SizeLimit
9. 日志:结构化消息模板,无 `Console.WriteLine`,无敏感信息
10. 迁移:未修改历史迁移,未手改生产库
11. 测试:业务变更有配套测试;缺陷修复有回归测试
12. 契约:API 变更同 PR 更新 OpenAPI/proto;破坏性变更已评审

## 项目脚手架

初始化新 .NET 服务端项目时:

1. 确认(未指定必须先询问):部署形态(单体/微服务)、数据库类型(mysql/postgresql)、API 风格(Controller/Minimal API)。
2. .NET 10 LTS;按 `rules/dotnet-server/practices.md`「解决方案结构」建 `Api/Application/Domain/Infrastructure/Shared` 五项目 + tests。
3. `.csproj` 启用 Nullable、ImplicitUsings、TreatWarningsAsErrors、锁文件;多项目配 `Directory.Build.props` + Central Package Management。
4. 落地骨架:统一响应结构、`IExceptionHandler` 全局异常处理、RequestId 中间件、健康检查(`/healthz` `/readyz`)、Serilog 结构化日志、Options Pattern 配置校验。
5. 首个业务模块按 Application/Domain/Infrastructure 分层打样,配 xUnit 测试项目。
