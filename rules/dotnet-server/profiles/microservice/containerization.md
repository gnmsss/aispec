# rules/dotnet-server/profiles/microservice/containerization.md

## Skill 协作
1. `$dotnet-server-coding-guide` 在识别到容器化部署、Docker 构建、Kubernetes 运行场景时加载本规则。
2. `$task-router` 在容器化与 K8s 部署任务中路由到本规则。

---

## 文档目标
1. 定义微服务容器化部署、Docker 镜像构建、Kubernetes 运行约束。
2. 非容器化部署的项目可跳过本文件。

---

## Docker 镜像规范（MUST）

1. 必须使用多阶段构建（Multi-Stage Build），最终镜像基于 `mcr.microsoft.com/dotnet/aspnet` 运行时镜像。
2. 禁止在最终镜像中包含 SDK 或构建工具，仅保留运行时依赖和发布产物。
3. 必须使用 `dotnet restore` + `dotnet publish -c Release` 确保可复现构建。
4. 镜像标签必须包含版本号或 Git commit hash，禁止仅使用 `latest`。
5. Dockerfile 必须纳入版本控制，与服务代码同仓管理。
6. 镜像构建必须在 CI 中自动执行，禁止依赖开发者本地构建并手动推送。
7. 应用代码 MUST 由非 root 用户运行。

### .NET 多阶段 Dockerfile 示例
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY Directory.Build.props Directory.Packages.props ./
COPY src/OrderService.Api/*.csproj src/OrderService.Api/
COPY src/OrderService.Application/*.csproj src/OrderService.Application/
COPY src/OrderService.Domain/*.csproj src/OrderService.Domain/
COPY src/OrderService.Infrastructure/*.csproj src/OrderService.Infrastructure/
RUN dotnet restore src/OrderService.Api/OrderService.Api.csproj
COPY src/ src/
RUN dotnet publish src/OrderService.Api/OrderService.Api.csproj \
    -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0
RUN addgroup --system app && adduser --system --ingroup app app
WORKDIR /app
COPY --from=build /app/publish .
USER app
EXPOSE 8080
ENTRYPOINT ["dotnet", "OrderService.Api.dll"]
```

### SHOULD
1. 定期扫描镜像漏洞（如 `trivy`、`grype`），高危漏洞阻断发布。
2. 基础镜像定期更新，跟进安全补丁。
3. 使用 `.dockerignore` 排除不必要的文件（`bin/`、`obj/`、`.git`、`tests/`）。

检查方式：Dockerfile 审查 + CI 构建验证
阻断级别：阻断合并

---

## Kubernetes 运行约束（MUST）

### 探针配置
1. 必须配置 `livenessProbe`（存活探针）：检测进程是否死锁/卡住，失败则重启容器。
2. 必须配置 `readinessProbe`（就绪探针）：检测服务是否可接受流量，失败则摘除流量。
3. 推荐配置 `startupProbe`（启动探针）：.NET 服务启动时需 JIT 预热时使用，避免存活探针误判。
4. 探针端点必须使用应用提供的 `/healthz` 和 `/readyz`（推荐 `Microsoft.Extensions.Diagnostics.HealthChecks`）。
5. 探针超时和间隔必须合理配置（建议：`initialDelaySeconds` ≥ 5s，`periodSeconds` 5-10s，`timeoutSeconds` 3-5s）。

### 资源限制
1. 每个容器必须设置 `resources.requests` 和 `resources.limits`（CPU + 内存）。
2. `requests` 与 `limits` 的比值建议 1:2 至 1:4，禁止设置过大的 limits 导致节点资源碎片。
3. .NET 服务内存 limits 必须考虑 GC 堆上限（`GCHeapHardLimit`），建议预留 20% 缓冲给非托管内存。
4. 禁止以 root 用户运行容器（`runAsNonRoot: true`）。

### 副本与调度
1. 生产环境必须至少 2 个副本，禁止单实例部署。
2. 必须配置 `podAntiAffinity`，确保同一服务的副本分布在不同节点。
3. 必须配置 `PodDisruptionBudget`（PDB），确保滚动更新或节点维护时至少有一个副本可用。

检查方式：K8s 配置审查
阻断级别：阻断合并

---

## 优雅停机与 K8s 集成（MUST）

1. `terminationGracePeriodSeconds` 必须 ≥ 应用优雅停机超时（参见 `common/concurrency-and-resource.md`）。
2. 容器收到 `SIGTERM` 后，ASP.NET Core 进程必须执行优雅停机流程（停止接收新请求 → 排空在途请求 → 关闭连接池 → 释放资源）。
3. `preStop` hook 推荐 sleep 3-5 秒，等待 Service Endpoint 摘除完成后再开始停机。
4. ASP.NET Core 默认支持 `IHostApplicationLifetime` 管理停机；后台任务使用 `IHostedService.StopAsync()` 执行清理。

检查方式：K8s 配置审查 + 集成测试
阻断级别：阻断合并
