# rules/dotnet-server/common/code-style.md

## 命名与组织
1. 命名空间与目录结构保持一致，禁止手动修改 `<RootNamespace>` 导致命名空间与物理路径不匹配。
2. 类名、方法名、属性名使用 PascalCase；参数名、局部变量使用 camelCase；私有字段使用 `_camelCase`。
3. 接口名以 `I` 前缀（如 `IUserService`），禁止使用 `Impl` 后缀命名实现类（如用 `UserService` 而非 `UserServiceImpl`）。
4. 异步方法必须以 `Async` 后缀命名（如 `GetUserAsync`），除非是 Controller Action 或 Minimal API Handler。
5. 文件命名必须体现职责，一个文件一个顶级类型（class/interface/enum/record），禁止 `Util.cs`、`Common.cs`、`Misc.cs` 等模糊命名。
6. 常量使用 PascalCase，禁止使用 `ALL_UPPER_CASE`；枚举值使用 PascalCase。

## 注释规范

### MUST
1. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
2. 所有公开的类型、方法、属性、接口必须有 XML 文档注释（`/// <summary>`），说明"做什么"和"为什么"。
3. Entity/Model/DTO 类的所有属性必须有 XML 文档注释，说明字段含义与约束。持久化实体属性注释需包含数据库列含义。
4. 非公开但逻辑复杂的方法（超过 30 行或包含分支、循环、并发）必须在方法头添加中文注释。
5. 复杂业务逻辑、非直觉的条件判断、临时方案（workaround）必须行内注释说明背景和原因。
6. 禁止无意义注释（如 `// 创建用户` 后面跟 `CreateUser()`），注释必须提供代码本身未表达的信息。
7. 接口注释必须说明实现方的职责约束和预期行为契约。

### SHOULD
1. TODO/FIXME 注释必须附带责任人和预计回收时间（如 `// TODO(zhangsan): 2026-04 迁移到新接口`）。
2. 注释随代码同步更新；代码逻辑变更后，对应注释必须同步修改，禁止过期注释残留。
3. 项目级 README 或 `doc.md` 说明项目的整体职责和主要用法。

检查方式：Roslyn 分析器（`SA1600` 系列）+ 人工审查
阻断级别：阻断合并

## 调试代码清理

### MUST
1. 禁止将 `Console.WriteLine`、`Debug.WriteLine`、`Trace.WriteLine` 等调试打印提交到主分支；所有日志输出必须通过项目统一的结构化日志组件（参见 `common/observability.md`）。
2. 禁止将 `Debugger.Break()`、`Debugger.Launch()` 等调试指令提交到主分支。
3. 禁止将 `#if DEBUG` 包裹的临时调试代码提交到主分支（正式的条件编译除外，需注释说明）。
4. CI 阶段通过 Roslyn 分析器检测并阻断调试代码残留：
   - 推荐使用 `.editorconfig` 配合分析器规则禁止 `Console.Write*` 等方法。
   - 配置示例（`.editorconfig`）：
   ```ini
   # 禁止使用 Console 类
   dotnet_diagnostic.CA2241.severity = error
   ```
5. 开发环境允许临时使用调试打印，但提交前必须清理；`git diff` 阶段应自查。

检查方式：Roslyn 分析器 + CI 阻断
阻断级别：阻断合并

## 分层编码要求
1. 分层依赖必须单向：`Controller/Endpoint -> Service/Application -> Repository/Infrastructure`，禁止反向依赖和循环依赖。
2. `Controller/Endpoint` 只负责协议适配：请求解析、参数校验、调用 Service、响应映射。
3. `Service/Application` 负责用例编排、事务边界、幂等策略、领域规则与权限策略。
4. `Repository/Infrastructure` 只负责数据访问与持久化映射，不承载业务决策、鉴权策略或流程编排。
5. 启动层（`Program.cs` / `Startup`）只做组装与生命周期管理，禁止承载业务逻辑。
6. 禁止在 Controller、Program.cs、Repository 之间跨层写业务捷径代码。

## 分层边界细则
1. `Controller/Endpoint` 禁止直接访问 `DbContext`、缓存客户端、对象存储客户端、消息中间件客户端。
2. `Controller/Endpoint` 禁止直接引用 Repository，必须经由 Service 调用。
3. `Service` 禁止依赖 ASP.NET Core HTTP 框架类型（`HttpContext`、`HttpRequest`）和协议层 DTO。
4. `Service` 不得直接编写 LINQ to SQL 或 EF Core 查询细节，数据访问必须通过 Repository。
5. `Repository` 禁止处理业务状态机、业务分支决策、跨聚合用例编排。
6. `Repository` 可以返回持久化结果，但对外暴露应保持稳定契约，不得泄露 EF Core 内部类型（如 `IQueryable` 不应泄露到 Service 层）。
7. `Domain` 模型与规则禁止依赖 Controller、Repository、Infrastructure 的具体实现。

## 模型与 DTO 约束
1. 协议层 DTO（Request/Response）仅用于 Controller/Endpoint，禁止下沉到 Service、Repository。
2. 持久化实体（Entity）用于数据库映射，作为数据库结构映射入口。
3. 临时读模型（Query DTO / Projection）仅用于统计分析、多表聚合查询，禁止用于常规 CRUD 写入。
4. 领域对象用于表达业务语义，禁止直接透传持久化实体到外部 API。
5. 不同层模型转换必须显式实现（推荐 Mapster 或手动映射），禁止在单个类上混用多层语义标注（如同时标注 `[JsonPropertyName]` 和 `[Column]`）。

## 事务、错误与日志边界
1. 事务边界定义在 Service 层，Repository 仅执行事务上下文内数据操作。
2. 错误处理遵循"内部系统异常记录、对外业务错误映射"原则。
3. Controller/Endpoint 负责将异常统一映射为稳定响应结构，禁止散落式手写映射逻辑（应通过 ExceptionHandler 中间件统一处理）。
4. Repository 返回异常时必须携带必要上下文并保留根因（使用 `throw` 而非 `throw ex`）。
5. 日志记录应在边界层和关键失败点进行，避免同一异常在多层重复记录。

## 代码可维护性
1. 每个方法只做一件事，避免超长方法和超深嵌套分支。
2. 公共代码先在模块内部复用，稳定后再考虑提升到共享类库。
3. 异步和资源生命周期必须在代码中可读可验证（`CancellationToken` 透传、`IDisposable`/`IAsyncDisposable` 实现、关闭顺序）。
4. 对外行为变化必须有测试覆盖：至少覆盖成功路径、参数错误、下游失败、超时或取消场景。

## 分层测试建议
1. Service 层优先单元测试，使用 mock/fake（推荐 Moq 或 NSubstitute）验证用例编排和领域规则。
2. Repository 层优先集成测试，验证 EF Core 查询行为、事务一致性与索引假设（推荐使用 Testcontainers）。
3. Controller/Endpoint 层使用集成测试（`WebApplicationFactory`），验证协议、状态码、错误映射和响应结构。
