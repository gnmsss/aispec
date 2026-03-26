# rules/dotnet-server/profiles/microservice/communication-and-contracts.md

## Skill 协作
1. `$dotnet-server-coding-guide` 在识别到微服务间通信、契约定义、gRPC/HTTP 调用场景时加载本规则。
2. `$task-router` 在跨服务通信任务中路由到本规则。

---

## 文档目标
1. 定义微服务间契约治理、通信协议选型、同步调用规范。
2. 异步通信参见 `messaging.md`；限流熔断参见 `resilience.md`；服务注册参见 `service-discovery.md`。

---

## 1. 契约治理（MUST）

1. 每个服务只维护一套对外契约源（OpenAPI 或 Proto），多调用方复用同一契约。
2. 契约必须版本化（如 `v1`、`v2`），破坏性变更必须升级大版本。
3. 契约变更发布前必须完成兼容性检查和消费方影响评估。
4. 禁止跨服务共享 EF Core DbContext、Entity 或数据库表结构对象。
5. 契约文件必须纳入版本控制，与服务代码同仓管理。
6. 契约变更必须附变更日志（CHANGELOG），说明新增、废弃、删除的字段和接口。
7. ASP.NET Core 服务的 Swagger 文档必须保持与代码同步（推荐 `Swashbuckle.AspNetCore` 或 `NSwag`）。
8. 共享契约类型（DTO/Event）MUST 发布为独立 NuGet 包（如 `OrderService.Contracts`），禁止跨服务直接引用对方的内部项目。

检查方式：人工审查 + 契约兼容性检查工具（如 `oasdiff`、`buf breaking`）
阻断级别：阻断合并

---

## 2. 通信协议选型（MUST）

### 选型矩阵

| 场景 | 推荐协议 | 原因 |
|------|---------|------|
| 服务间内部同步调用 | **gRPC**（Proto3） | 高性能、强类型、`Grpc.Net.Client` 原生支持 |
| 对外 API（面向前端/第三方） | **HTTP/JSON**（RESTful） | 生态兼容性好、调试便捷 |
| 需要浏览器直接调用内部服务 | **HTTP 网关** 或 **gRPC-Web** | 浏览器不支持原生 gRPC |
| 文件上传/下载 | **HTTP** | gRPC 不适合大文件流式传输 |

### MUST
1. 微服务间内部通信推荐使用 **gRPC + Proto3**（`Grpc.AspNetCore` + `Grpc.Net.Client`），或 **HTTP/JSON**（`IHttpClientFactory` + 类型化客户端）。
2. 对外 API（面向浏览器/小程序/第三方）使用 **HTTP/JSON（RESTful）**。
3. 同一服务可同时提供 gRPC 和 HTTP 端口，但两者必须共享同一套 `Application` 层，禁止 transport 层存在逻辑分叉。
4. 协议选型必须在服务设计阶段确定并记录，禁止开发过程中随意切换。

### 多语言服务协作（MUST）
1. 跨语言服务间通信必须使用 **gRPC + Proto3** 作为统一协议，禁止依赖语言特定的序列化格式。
2. Proto 文件作为跨语言唯一契约源，由服务提供方维护，消费方通过代码生成获取客户端。
3. 禁止手写跨语言客户端代码，必须通过 Proto 代码生成工具自动生成。

### SHOULD
1. 服务间 HTTP 调用推荐使用 `IHttpClientFactory` + Polly 弹性策略 + 类型化客户端封装。
2. Proto 文件使用 `buf` 工具链管理（lint、breaking change 检测、代码生成）。

检查方式：架构评审 + Proto lint（`buf lint`）
阻断级别：阻断合并

---

## 3. 同步调用规范（MUST）

1. 必须配置超时、重试、熔断、限流，且参数可配置（熔断/限流细则参见 `resilience.md`）。
2. 重试必须幂等，非幂等接口不得无条件重试。
3. 重试策略必须包含：最大重试次数（建议 ≤ 3）、退避策略（指数退避 + 随机抖动）、可重试状态码白名单。
4. 服务间调用必须透传 trace 上下文（`traceparent` Header / gRPC metadata）和请求标识（`requestId`）。
5. 下游不可用时必须有降级策略或显式失败策略，禁止无限等待。
6. 超时设置分层：全局默认 → 服务级 → 接口级，接口级优先。

### IHttpClientFactory + Polly 示例
```csharp
builder.Services
    .AddHttpClient<IUserServiceClient, UserServiceClient>(client =>
    {
        client.BaseAddress = new Uri("http://user-svc:5000");
        client.Timeout = TimeSpan.FromSeconds(10);
    })
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt))
            + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100))))
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

### 超时传播（MUST）
1. 上游传入的 deadline 必须向下游传播，禁止下游超时比上游更长。
2. 调用链每一跳必须预留处理时间。
3. gRPC 调用推荐使用 `CallOptions` 传递 deadline。

检查方式：代码审查 + 集成测试
阻断级别：阻断合并

---

## 4. 数据一致性策略（MUST）

1. 跨服务禁止依赖本地数据库事务保证一致性。
2. 优先使用最终一致性方案（事件驱动、补偿事务、Outbox）。
3. 必须明确写入顺序、重试语义、补偿边界与失败回滚策略。
4. Outbox 模式：业务写入与 Outbox 消息在同一本地事务中；独立调度器轮询投递；消费方幂等。推荐使用 MassTransit 内置的 Transactional Outbox。
5. Saga 模式：每个参与方提供正向+补偿操作；补偿必须幂等；必须定义超时和人工介入机制。推荐使用 MassTransit Saga（State Machine）。

检查方式：架构评审 + 集成测试
阻断级别：阻断合并
