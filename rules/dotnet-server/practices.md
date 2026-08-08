# .NET 服务端场景实践(按需加载)

硬约束与红线见 `baseline.md`。

## 解决方案结构(模块化单体)

```text
MyApp/
├── src/
│   ├── MyApp.Api/                  # 启动项目:Program.cs、Controllers(按 Admin/Client 作用域分目录)、
│   │   │                          # Middlewares(GlobalExceptionHandler、RequestId)、Auth(各作用域独立 Handler)、Dto
│   ├── MyApp.Application/          # 用例编排:按业务模块分目录(Users/、Orders/),含 IXxxService + XxxService + Validators
│   ├── MyApp.Domain/               # Entities、Exceptions(BusinessException 基类 + 各域异常)、ErrorCodes、Repository 接口
│   ├── MyApp.Infrastructure/       # Data(AppDbContext、Configurations、Repositories、Queries 投影 DTO、Migrations)、
│   │   │                          # Caching(CacheKeys 集中定义)、Storage、Extensions(DI 注册扩展)
│   └── MyApp.Shared/               # Options 强类型配置、常量(仅技术组件)
└── tests/                          # 单元测试 + 集成测试项目
```

1. Admin 与 Client 等不同作用域使用独立认证 Handler 与独立 Controller 目录,禁止同一处 if 分支混合鉴权。
2. 模块间禁止直接调用对方 Repository;跨模块协作走 Application 层接口。

## API 契约

1. HTTP 契约用 OpenAPI(.NET 内置 `Microsoft.AspNetCore.OpenApi` 或 Swashbuckle);gRPC 用 proto3 + `Grpc.AspNetCore`。
2. 契约主来源二选一(Contract-First / Code-First),API 变更与契约同 PR 更新;破坏性变更评审升版本。
3. Minimal API 与 Controller 风格项目内统一,不混用;端点分组用 `MapGroup` 并统一挂过滤器。

## 组件初始化与生命周期

1. 统一使用内置 DI:生命周期约定 `DbContext` 为 Scoped、无状态服务 Singleton、用例服务 Scoped;注册收敛到 `Infrastructure/Extensions` 的 `AddXxx` 扩展方法,`Program.cs` 只做调用编排。
2. 健康检查用 `AddHealthChecks`:`/healthz`(存活)与 `/readyz`(就绪,检查 DB/Redis 等关键依赖,`AddDbContextCheck`、`AddRedis`)。
3. 优雅停机:依赖 Host 的 `IHostApplicationLifetime`;`ShutdownTimeout` 可配置;后台任务用 `BackgroundService` 并响应 `stoppingToken`。
4. 组件日志脱敏,初始化失败 fail fast;可选组件降级需记录日志。

## 数据库访问(EF Core)

1. 主 ORM 用 EF Core(EF Core 10);复杂查询/性能敏感可补充 Dapper,但连接管理必须统一;禁止两套以上 ORM。
2. `AddDbContext`/`AddDbContextPool` 注册;实体配置用 Fluent API 独立文件(`IEntityTypeConfiguration<T>`);显式配置 `CommandTimeout` 与 `EnableRetryOnFailure`。
3. 查询:只读场景必须 `AsNoTracking()`;LINQ 用 `.Select()` 投影所需字段,禁止取整实体只用部分字段;关联加载用 `Include`/`ThenInclude` 或批量查询,禁止 N+1。
4. 批量操作用 `ExecuteUpdate`/`ExecuteDelete` 或 `EFCore.BulkExtensions`,单批限制大小。
5. 事务边界在 Application 层:同一 Scoped `DbContext` 天然共享事务,跨资源用 `IDbContextTransaction`;显式指定 `IsolationLevel`;事务内禁止远程调用。
6. 迁移:只用 EF Core Migrations,禁止修改历史迁移、禁止手改生产库;结构变更遵循 `rules/database.md`;生产迁移独立执行(禁止启动时自动 Migrate)。
7. 慢查询:`DbCommandInterceptor` 检测,阈值可配置(默认 200ms),Warning 级记录脱敏 SQL + 耗时 + 来源;生产禁开全量 SQL 日志。

## 缓存

1. 分布式缓存用 Redis(`IDistributedCache` 或 StackExchange.Redis 直连,项目内统一);进程内缓存用 `IMemoryCache`(必须设 `SizeLimit`)或 .NET 9+ `HybridCache`。
2. 键集中定义(`Infrastructure/Caching/CacheKeys.cs`),格式 `{服务}:{业务域}:{标识}`;全部设 TTL 加随机偏移。
3. Cache-Aside 为默认模式:先写库后删缓存,删除失败有补偿;穿透用空值缓存/布隆过滤器;击穿高热点键用锁或逻辑过期;强一致数据(金额、库存)禁止以缓存为数据源。

## 定时任务与后台 Worker

1. 常驻任务用 `BackgroundService`;复杂调度用 Quartz.NET 或 Hangfire,项目内统一选型。
2. 多实例定时任务必须分布式锁(Redis/数据库),锁带超时,标识含任务名+周期;任务必须幂等,执行结果持久化。
3. 失败告警 + 重试 ≤ 3 次带退避;长任务用 `CancellationToken` 超时取消。

## 文件存储

1. 统一对象存储(MinIO/OSS/S3),SDK 封装在 Infrastructure,业务经接口调用;禁止存本地磁盘。
2. 上传校验大小 + MIME/文件头白名单;存储名 UUID 重命名;大文件分片;元信息入库。
3. 访问分级:公开(CDN)/受限(签名 URL ≤ 30min)/敏感(SSE + 短签名 + 审计),不同级别不同 Bucket。

## 可观测性

1. 指标与追踪统一 OpenTelemetry(`AddOpenTelemetry().WithMetrics().WithTracing()`),W3C Trace Context 透传;Prometheus 导出绑定管理端口。
2. RED 指标 + 下游依赖成功率/耗时 + runtime 指标(GC、线程池、连接池)。
3. 告警必配:错误率 >1%、P99 >1s、实例不可用、线程池饥饿、连接池耗尽;分级 P0/P1/P2 并指定负责人。

## 性能

1. 热路径避免高频分配:`Span<T>`/`Memory<T>`、`ArrayPool<T>`、`StringBuilder`;JSON 用 `System.Text.Json`(源生成器优先),禁止引入 Newtonsoft 处理新代码热路径。
2. 响应时间目标 P95 ≤ 200ms、P99 ≤ 500ms;耗时操作异步化返回任务 ID;大响应启用压缩中间件。
3. 性能敏感模块写 BenchmarkDotNet 基准;线上排查用 `dotnet-counters`/`dotnet-trace`/`dotnet-dump`。

## 测试与发布

1. Application 层单元测试(xUnit + NSubstitute/Moq,覆盖正常/边界/异常);Repository 集成测试(Testcontainers 起真实数据库);API 层用 `WebApplicationFactory` 做接口测试。
2. 修复缺陷必须补回归测试;发布支持健康检查、优雅停机、失败回滚。

## 微服务差异(microservice profile)

1. 每服务独立解决方案或 monorepo 明确边界;禁止共享实体类库,服务间只通过契约(OpenAPI/proto)通信。
2. 同步调用统一 gRPC 或 HTTP;跨服务一致性用 Saga/Outbox(如 MassTransit/CAP),禁止分布式大事务。
3. 弹性:`Microsoft.Extensions.Resilience`(Polly v8)配置重试、熔断、超时,超时逐层递减。
4. 消息:MassTransit/CAP 统一选型;消费幂等 + 死信队列 + 积压告警;消息头透传 trace 上下文。
5. 服务发现与配置中心按平台统一(K8s Service/Consul/Nacos);功能开关热更新可回滚。
6. 容器:多阶段构建(`mcr.microsoft.com/dotnet/aspnet` 运行时镜像)、非 root 运行;K8s 配置探针与资源限制;发布走灰度。
