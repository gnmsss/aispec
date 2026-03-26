# rules/dotnet-server/profiles/microservice/resilience.md

## Skill 协作
1. `$dotnet-server-coding-guide` 在识别到限流、熔断、降级场景时加载本规则。
2. `$task-router` 在服务韧性与容错任务中路由到本规则。

---

## 文档目标
1. 定义微服务场景下的限流、熔断、降级约束。

---

## 限流（MUST）

1. 面向外部流量的 API 必须配置限流。
2. 限流算法推荐：**令牌桶**（平滑突发）或 **滑动窗口**（精确计数）。
3. 限流维度必须可配置：全局 / 接口级 / 用户级 / IP 级。
4. 限流触发后返回标准响应（HTTP `429 Too Many Requests`），包含 `Retry-After` 提示。
5. 限流阈值必须可配置，禁止硬编码。
6. ASP.NET Core 7+ 推荐使用内置 `Microsoft.AspNetCore.RateLimiting` 中间件；分布式限流推荐 Redis + Lua 脚本。

### ASP.NET Core 限流示例
```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 0;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();
```

检查方式：架构评审 + 集成测试
阻断级别：阻断合并

---

## 熔断（MUST）

1. 服务间调用必须配置熔断器。
2. 熔断器支持三态：**关闭** → **打开** → **半开**。
3. 触发条件可配置：错误率阈值（如 > 50%）、慢调用率阈值（如 > 70%）、最小样本数。
4. 熔断打开期间必须执行降级策略，禁止直接抛异常给调用方。
5. 半开阶段允许少量探测请求，探测成功则恢复，失败则重新打开。
6. .NET 项目推荐使用 **Polly**（`Microsoft.Extensions.Http.Polly`）或 .NET 8+ 内置的 `Microsoft.Extensions.Resilience`。

### Polly 熔断示例
```csharp
builder.Services
    .AddHttpClient<IUserServiceClient, UserServiceClient>()
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)))
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt))));
```

检查方式：集成测试（熔断触发与恢复）
阻断级别：阻断合并

---

## 降级（MUST）

1. 核心链路必须定义降级策略：
   - **静态降级**：返回预设默认值或缓存数据。
   - **功能降级**：关闭非核心功能，保障核心流程。
   - **流量降级**：拒绝低优先级请求，保障高优先级。
2. 降级触发和恢复必须有日志和指标记录。
3. 熔断器打开时的 fallback 函数必须有独立实现，禁止在 fallback 中再次调用故障服务。

### Polly Fallback 示例
```csharp
var fallbackPolicy = Policy<UserDto>
    .Handle<HttpRequestException>()
    .Or<BrokenCircuitException>()
    .FallbackAsync(async ct =>
    {
        logger.LogWarning("user_service_degraded, returning cached data");
        var cached = await cache.GetAsync<UserDto>($"user:{userId}");
        return cached ?? new UserDto { Id = userId, Username = "unknown", IsDegraded = true };
    });
```

### SHOULD
1. 限流、熔断、降级配置优先通过配置中心动态下发，支持运行时调整。
2. 相关指标纳入监控仪表盘，实时可观测。

检查方式：架构评审
阻断级别：阻断合并
