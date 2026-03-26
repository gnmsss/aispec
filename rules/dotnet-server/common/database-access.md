# rules/dotnet-server/common/database-access.md

## 数据库类型约束
1. 服务必须明确数据库类型，仅允许 `mysql` 或 `postgresql` 两种选择。
2. 数据库类型必须与配置文件中的 `Database:Type` 一致，不得在代码中隐式推断。
3. 若需求中出现数据库接入但未指定类型，必须先反馈并由用户选择后再继续实现。
4. MySQL 使用 `Pomelo.EntityFrameworkCore.MySql`，PostgreSQL 使用 `Npgsql.EntityFrameworkCore.PostgreSQL`。

## ORM 选型
1. 默认使用 EF Core 作为主 ORM，适用于常规 CRUD 和事务操作。
2. 复杂查询或性能敏感场景允许使用 Dapper 作为补充，但必须在同一项目中统一数据库连接管理。
3. 禁止同时使用两套以上 ORM 框架，避免连接和事务管理混乱。

## 数据模型约束
1. 必须建立持久化实体（Entity）映射数据库结构，作为常规查询与写入的默认模型。
2. 常规查询（单表或稳定业务查询）必须使用持久化实体，禁止长期使用匿名类型或 `Dictionary` 直接承载结果。
3. 分析统计、报表、多表联合聚合等场景可使用投影 DTO（Projection / Query DTO）替代持久化实体。
4. 投影 DTO 仅用于查询结果承载，不得直接用于写入、更新或替代持久化实体的领域语义。
5. 新增投影 DTO 必须标注用途与作用域，并按职责独立文件命名（如 `OrderStatDto.cs`、`UserReportItemDto.cs`）。

## EF Core 配置规范
1. `DbContext` 注册为 `Scoped`，使用 `AddDbContext<T>` 或 `AddDbContextPool<T>`。
2. 实体配置推荐使用 Fluent API（`IEntityTypeConfiguration<T>`），禁止仅依赖 Data Annotations 做复杂映射。
3. 实体配置必须独立文件（如 `UserConfiguration.cs`），禁止在 `OnModelCreating` 中堆积所有配置。
4. 必须显式配置 `CommandTimeout`：
   ```csharp
   options.UseMySql(connectionString, serverVersion, opt =>
   {
       opt.CommandTimeout(30);
       opt.EnableRetryOnFailure(maxRetryCount: 3);
   });
   ```
5. 启用连接弹性（`EnableRetryOnFailure`），配置最大重试次数和重试间隔。

## 查询与写入规范
1. LINQ 查询必须使用投影（`.Select()`），禁止查询整个实体后仅使用部分字段（等同于 `SELECT *`）。
2. 参数化查询是强制要求（EF Core 默认参数化，Dapper 必须使用参数对象），禁止字符串拼接 SQL。
3. 批量写入和批量更新必须限制单批大小，推荐使用 `EFCore.BulkExtensions` 或 `ExecuteUpdate` / `ExecuteDelete`（EF Core 7+）。
4. 查询必须使用 `AsNoTracking()` 除非需要跟踪变更（只读场景性能优化）。
5. 禁止在循环中执行单条数据库操作（N+1 问题），必须使用 `Include` / `ThenInclude` 或批量查询。

## 事务边界
1. 事务边界在 Service 层定义，Repository 只执行事务上下文内操作。
2. 单次事务应短小，禁止在事务中执行远程调用或不确定时延逻辑。
3. 必须明确隔离级别（通过 `IsolationLevel` 参数），避免隐式锁冲突。
4. 跨多个 Repository 的事务使用 `DbContext` 共享（Scoped 生命周期自动保证）或 `IDbContextTransaction`。
5. 示例：
   ```csharp
   await using var transaction = await _dbContext.Database
       .BeginTransactionAsync(IsolationLevel.ReadCommitted, cancellationToken);
   try
   {
       // 多个 Repository 操作...
       await _dbContext.SaveChangesAsync(cancellationToken);
       await transaction.CommitAsync(cancellationToken);
   }
   catch
   {
       await transaction.RollbackAsync(cancellationToken);
       throw;
   }
   ```

## 数据库迁移
1. 数据库结构变更必须使用 EF Core Migrations，禁止手动修改生产数据库。
2. 迁移脚本必须纳入版本控制，迁移文件命名自动生成即可。
3. 禁止修改历史迁移脚本，变更必须通过新增迁移交付。
4. 数据库结构或初始化数据变更必须遵守 `rules/database/database.md`。

## 数据库操作日志（MUST）
1. 所有数据库操作必须可追踪，通过 EF Core 的日志机制或拦截器记录操作日志，至少包含：操作类型、目标表、执行耗时、是否成功。
2. 慢查询阈值必须可配置（建议默认 200ms），超过阈值的查询必须以 `Warning` 级别记录日志并标记 `SlowQuery: true`。
3. 慢查询日志必须与正常应用日志输出到同一日志通道（不单独分离文件），通过日志级别和标记字段区分。
4. 慢查询日志须包含：SQL 语句（脱敏后）、执行耗时、影响行数、调用来源。
5. 推荐使用 EF Core 的 `IInterceptor`（`DbCommandInterceptor`）或 `LogTo` 配合 `ILogger` 实现慢查询检测和日志记录。
6. 开发环境允许开启全量 SQL 日志（`EnableSensitiveDataLogging` + `EnableDetailedErrors`），生产环境仅记录慢查询和错误查询。

检查方式：慢查询日志审查 + EXPLAIN 审查 + 代码审查
阻断级别：阻断合并
