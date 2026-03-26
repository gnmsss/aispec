# rules/dotnet-server/profiles/microservice/config-center.md

## Skill 协作
1. `$dotnet-server-coding-guide` 在识别到配置中心、动态配置、热更新场景时加载本规则。
2. `$task-router` 在配置管理任务中路由到本规则。

---

## 文档目标
1. 定义微服务场景下配置中心选型、动态配置、热更新约束。
2. 静态配置文件规范参见 `common/configuration.md`，本文件聚焦运行时动态配置。

---

## 配置中心选型（MUST）

| 方案 | 适用场景 | .NET 客户端 |
|------|---------|------------|
| **Nacos** | 配置+注册一体化需求 | `nacos-sdk-csharp` |
| **Consul KV** | 已选 Consul 做服务注册 | `Consul` NuGet 包 + `IConfigurationBuilder` 扩展 |
| **etcd** | Kubernetes 生态 / 已有 etcd 基础设施 | `dotnet-etcd` |
| **Apollo** | 大规模微服务、多环境多集群 | `Com.Ctrip.Framework.Apollo.Configuration` |
| **Azure App Configuration** | Azure 生态 | `Microsoft.Azure.AppConfiguration.AspNetCore` |

1. 项目必须选定唯一的配置中心方案，禁止混用。
2. 选型必须在架构设计阶段确定并记录。
3. 配置中心不可用时，服务必须能以本地 `appsettings.json` / 环境变量兜底启动，禁止因配置中心故障导致服务无法启动。
4. 配置中心客户端 MUST 注册为 `IConfigurationProvider`，融入 ASP.NET Core 标准配置体系。

检查方式：架构评审
阻断级别：阻断合并

---

## 配置分类（MUST）

| 分类 | 示例 | 是否支持热更新 | 存放位置 |
|------|------|--------------|---------|
| **静态配置** | 数据库连接串、服务端口、日志级别 | 否（需重启） | `appsettings.json` / 环境变量 |
| **动态配置** | 限流阈值、熔断参数、功能开关 | 是（运行时生效） | 配置中心 |
| **敏感配置** | 数据库密码、API 密钥、证书 | 否 | 密钥管理服务（Vault / Azure Key Vault / K8s Secret） |

1. 配置必须按上述分类管理，禁止将敏感配置放入配置中心明文存储。
2. 动态配置的变更必须有审计日志（谁在什么时间改了什么值）。
3. 敏感配置必须通过密钥管理服务或环境变量注入，禁止硬编码或写入配置文件。
4. ASP.NET Core 的 `IOptionsMonitor<T>` 用于响应动态配置变更；`IOptionsSnapshot<T>` 用于 Scoped 场景。

检查方式：配置审查
阻断级别：阻断合并

---

## 热更新规范（MUST）

1. 支持热更新的配置项必须显式标注（代码注释或文档），禁止默认所有配置都支持热更新。
2. 热更新回调必须线程安全，更新过程中禁止出现中间态影响业务逻辑。推荐使用 `IOptionsMonitor<T>.OnChange()` 回调。
3. 热更新后必须记录日志：配置项名称、旧值、新值、生效时间。
4. 连接池参数（数据库、Redis）等涉及资源重建的配置，禁止热更新，必须重启生效。
5. 配置变更必须支持回滚：配置中心必须保留历史版本，可一键回退到上一版本。

### SHOULD
1. 动态配置变更支持灰度推送（先推送部分实例，观察后再全量）。
2. 关键配置变更触发告警通知相关负责人。

检查方式：代码审查 + 配置中心审计日志
阻断级别：阻断合并

---

## 功能开关（Feature Flag）（SHOULD）

1. 新功能上线推荐使用功能开关，支持运行时启停，降低发布风险。
2. 功能开关必须有明确的生命周期：创建 → 灰度 → 全量 → 清理代码。
3. 功能全量上线后，必须在下一个版本中清理开关代码，禁止长期保留无用开关。
4. 功能开关的状态必须可观测（监控面板可查看当前开启的开关列表）。
5. .NET 项目推荐使用 `Microsoft.FeatureManagement`（ASP.NET Core 原生集成）或配置中心 KV 存储。

检查方式：代码审查
阻断级别：告警记录
