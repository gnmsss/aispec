# rules/dotnet-server/index.md

## 目的
1. 统一 C#/.NET 服务端开发与交付标准，降低架构漂移和协作成本。
2. 采用"共性规则 + 场景规则"模式，避免重复和冲突。

## 适用范围
1. 适用于所有 C#/.NET 服务端代码：Web API、gRPC 服务、消息消费、后台任务、Worker Service。
2. 本规则默认高于个人编码习惯；若需例外，必须在评审中记录原因、边界、回收时间。

## 规则组成
1. `common`：所有 C#/.NET 服务端必须遵守。
2. `profiles/monolith`：单体应用额外规则。
3. `profiles/microservice`：微服务额外规则。
4. 跨端协作：`rules/frontend-backend-collaboration.md`（前后端契约与联调）。

## 适用方式
1. 单体应用：`common + profiles/monolith`。
2. 微服务：`common + profiles/microservice`（按需加载命中的文件，不必通读全部）。
3. 混合架构（同仓多服务）：每个可执行程序独立选择 profile，不得混用一套边界定义。

## Skill 协作（推荐）
1. 编写 C#/.NET 服务端代码时优先使用 `$dotnet-server-coding-guide`，按编码场景自动加载规则。
2. 跨域业务任务（涉及多个技术栈）使用 `$task-router` 自动分析并路由。
3. 规则维护优先使用 `$dotnet-server-rules-maintainer`。
4. 涉及前后端契约、联调、发布回滚时优先使用 `$frontend-backend-coding-guide`。

## 冲突优先级
1. 具体 profile 规则优先于 `common` 中同主题的描述。
2. 数据库变更规则以 `rules/database/database.md` 为准。
3. 前后端协作相关条款以 `rules/frontend-backend-collaboration.md` 为准。
4. 当规则冲突无法消解时，以"更严格、更可验证"的规则为准。

## 目录索引

### 通用规则（common）— 所有 C#/.NET 服务端必须遵守
1. `common/baseline.md` — 技术基线与基础工程要求
2. `common/code-style.md` — 命名、注释、分层编码、调试代码清理
3. `common/component-initialization.md` — 依赖注入、生命周期、健康检查
4. `common/api-design.md` — API 版本化、响应结构、契约治理
5. `common/error-handling.md` — 异常分类、传播、错误码治理
6. `common/observability.md` — 结构化日志、指标、链路追踪
7. `common/configuration.md` — 配置文件组织、环境变量、基础设施约束
8. `common/concurrency-and-resource.md` — 异步编程、资源生命周期、优雅停机
9. `common/database-access.md` — 数据模型、查询规范、事务边界
10. `common/security.md` — 输入校验、鉴权、审计、数据保护
11. `common/testing-and-release.md` — 测试要求、质量门禁、发布规范
12. `common/performance.md` — Profiling、内存管理、查询性能、连接池调优
13. `common/caching.md` — 缓存选型、键设计、TTL、穿透/击穿/雪崩防护、一致性
14. `common/file-storage.md` — 文件上传下载、对象存储（MinIO/OSS）、临时文件清理
15. `common/scheduled-tasks.md` — 后台任务、定时作业、分布式锁、幂等、失败处理
16. `common/forbidden.md` — 禁止事项汇总

### 单体应用规则（profiles/monolith）
17. `profiles/monolith/project-structure.md` — 单体应用目录结构与模块边界

### 微服务规则（profiles/microservice）— 按需加载命中文件
18. `profiles/microservice/project-structure.md` — 微服务目录结构与边界
19. `profiles/microservice/communication-and-contracts.md` — 服务间通信与契约治理
20. `profiles/microservice/config-center.md` — 配置中心与动态配置
21. `profiles/microservice/containerization.md` — 容器化部署与 Kubernetes 运行约束
22. `profiles/microservice/deployment-and-release.md` — 版本管理与灰度发布
23. `profiles/microservice/messaging.md` — 异步消息与消息队列
24. `profiles/microservice/resilience.md` — 限流、熔断与降级
25. `profiles/microservice/service-discovery.md` — 服务注册与发现

### 跨端协作
19. `rules/frontend-backend-collaboration.md` — 前后端契约、联调、发布回滚

### 配套模板 — 参见 `rules/templates/index.md`
- `templates/dotnet-server/pr-review-checklist.md` — C#/.NET 服务端 PR 评审清单
- `templates/exception-request-template.md` — 规范例外申请模板（通用）
