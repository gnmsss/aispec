# rules/dotnet-server/profiles/microservice/service-discovery.md

## Skill 协作
1. `$dotnet-server-coding-guide` 在识别到服务注册、服务发现、负载均衡场景时加载本规则。
2. `$task-router` 在服务注册与发现任务中路由到本规则。

---

## 文档目标
1. 定义微服务注册与发现、负载均衡的约束。

---

## 注册中心选型（MUST）

| 注册中心 | 适用场景 | .NET 客户端 |
|---------|---------|------------|
| **Consul** | 通用微服务 | `Consul` NuGet 包 + Steeltoe / 自定义 `IHostedService` |
| **etcd** | Kubernetes 生态 / 已有 etcd 基础设施 | `dotnet-etcd` |
| **Nacos** | 已有 Java/Spring Cloud 生态 | `nacos-sdk-csharp` |
| **Kubernetes Service** | 全容器化部署 | 零额外组件、DNS 服务发现 |

1. 项目必须选定唯一的服务注册与发现方案，禁止混用多套注册中心。
2. 选型必须在架构设计阶段确定并记录，变更需经架构评审。
3. 全容器化且使用 Kubernetes 的项目，允许使用 K8s Service + DNS 作为服务发现方案。

检查方式：架构评审
阻断级别：阻断合并

---

## 注册与注销（MUST）

1. 服务注册信息必须包含：服务名、实例地址、端口、协议类型（gRPC/HTTP）、版本号、健康检查端点。
2. 服务启动时必须主动注册，停止时必须主动注销（优雅停机阶段执行）。
3. 服务消费方禁止硬编码目标服务地址，必须通过服务发现获取实例列表。
4. 注册/注销逻辑 MUST 封装为 `IHostedService`，在 `StartAsync` 中注册、`StopAsync` 中注销。

### SHOULD
1. 注册信息中携带元数据标签（如 `env=prod`、`region=cn-east`），支持按标签路由。
2. 服务实例列表变更通过 watch/subscribe 实时感知，而非定时轮询。

检查方式：集成测试（注册/注销验证）
阻断级别：阻断合并

---

## 健康检查（MUST）

1. 必须配置两类检查：
   - **存活检查（Liveness）**：进程存活，失败则重启。
   - **就绪检查（Readiness）**：可接受流量，失败则摘除流量。
2. 健康检查端点建议独立（`/healthz`、`/readyz`），与业务路由分离。
3. 就绪检查必须验证核心依赖可用性（数据库、Redis、消息队列）。
4. ASP.NET Core MUST 使用 `Microsoft.Extensions.Diagnostics.HealthChecks` + `AspNetCore.HealthChecks.*` 社区包实现健康检查。

### 健康检查配置示例
```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "database")
    .AddRedis(redisConnectionString, name: "redis")
    .AddRabbitMQ(rabbitConnectionString, name: "rabbitmq");

app.MapHealthChecks("/healthz", new HealthCheckOptions
{
    Predicate = _ => false
});
app.MapHealthChecks("/readyz", new HealthCheckOptions
{
    Predicate = _ => true,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

检查方式：集成测试
阻断级别：阻断合并

---

## 负载均衡（MUST）

1. 客户端负载均衡必须支持至少一种策略（Round Robin / 加权轮询 / 最少连接）。
2. gRPC 客户端必须启用客户端负载均衡（`Grpc.Net.Client` 支持通过 `ServiceConfig` 配置），禁止所有请求打到同一实例。
3. 故障实例必须自动摘除：健康检查失败的实例在超时窗口后不再接收流量。
4. 使用 K8s Service 时，可依赖 K8s 内置的 kube-proxy 负载均衡。

### SHOULD
1. 多机房/多可用区部署时，优先路由到同区实例（亲和性路由）。

检查方式：架构评审 + 集成测试
阻断级别：阻断合并
