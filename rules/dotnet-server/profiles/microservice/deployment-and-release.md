# rules/dotnet-server/profiles/microservice/deployment-and-release.md

## Skill 协作
1. `$dotnet-server-coding-guide` 在识别到版本管理、发布流程、灰度发布场景时加载本规则。
2. `$task-router` 在部署与发布任务中路由到本规则。

---

## 文档目标
1. 定义微服务版本管理与灰度发布约束。

---

## 版本管理（MUST）

1. 服务版本号遵循语义化版本（SemVer：`MAJOR.MINOR.PATCH`）。
2. 多版本共存期间，服务注册信息中必须携带版本号，消费方可按版本路由。
3. 接口废弃必须走 Deprecation 流程：标记废弃 → 通知消费方 → 观察用量 → 下线。
4. 版本号 MUST 在 `Directory.Build.props` 或 `.csproj` 中声明，且与 Git tag / Docker 镜像标签一致。

检查方式：发布流程审查
阻断级别：阻断合并

---

## 灰度发布（SHOULD）

1. 推荐策略：
   - **金丝雀发布**：新版本接收小比例流量（如 5%），观察指标后逐步放量。
   - **蓝绿部署**：新旧版本并存，通过流量切换完成发布。
2. 灰度期间必须配置核心指标告警（错误率、延迟），异常自动回滚或人工介入。
3. K8s 环境推荐使用 `Argo Rollouts` 或 `Flagger` 实现自动化灰度发布。

### SHOULD
1. 发布前完成压力测试，对比性能基线（参见 `common/performance.md`）。
2. 回滚方案必须预先制定并验证可执行性。

检查方式：发布流程审查
阻断级别：告警记录

---

## 数据库迁移与发布协调（MUST）

1. 数据库迁移（EF Core Migrations）MUST 在应用部署前独立执行，禁止在应用启动时自动运行 `Database.Migrate()`。
2. 迁移脚本必须向后兼容：新版应用代码必须兼容旧版数据库 schema（至少一个版本窗口）。
3. 涉及不兼容 schema 变更时，必须采用多步发布策略：
   - 第一步：发布兼容新旧 schema 的代码。
   - 第二步：执行 schema 迁移。
   - 第三步：发布仅兼容新 schema 的代码。
4. 大表 DDL 变更必须评估锁影响，必要时使用在线 DDL 工具。

检查方式：发布流程审查
阻断级别：阻断合并

---

## CI/CD 流水线（MUST）

1. CI 流水线必须包含以下阶段：
   - **Build**：`dotnet build` 编译检查
   - **Test**：`dotnet test` + 覆盖率检查
   - **Security**：`dotnet list package --vulnerable` 依赖漏洞扫描
   - **Publish**：`dotnet publish -c Release` + Docker 镜像构建 + 标签
   - **Push**：镜像推送到 Registry
2. CD 流水线必须支持：
   - 自动部署到 staging 环境。
   - 人工审批后部署到 production 环境。
   - 一键回滚到上一个稳定版本。
3. 每次发布必须生成发布记录（Release Notes），包含变更内容、影响范围、回滚方案。

检查方式：CI/CD 配置审查
阻断级别：阻断合并

---

## 发布前检查清单（SHOULD）

1. 所有测试通过（单元测试 + 集成测试 + e2e 测试）。
2. 依赖漏洞扫描无高危漏洞。
3. 数据库迁移脚本已独立验证。
4. 配置变更已同步（环境变量、配置中心）。
5. 监控告警规则已更新。
6. 回滚方案已确认可执行。
7. 相关团队已通知。

检查方式：发布流程审查
阻断级别：告警记录
