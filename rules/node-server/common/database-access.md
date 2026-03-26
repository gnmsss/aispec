# rules/node-server/common/database-access.md

## ORM 选型与使用

### MUST
1. 项目必须选定统一的数据访问方案（Prisma / TypeORM / Drizzle / Knex），禁止同类场景混用多套 ORM。
2. ORM 选型必须在项目初期评审确定，中途更换须提交迁移方案并经评审。

| ORM | 适用场景 | 特点 |
|-----|---------|------|
| **Prisma** | 类型安全优先、Schema-first | 自动生成类型、迁移管理完善 |
| **TypeORM** | 装饰器风格、NestJS 深度集成 | ActiveRecord/DataMapper 双模式 |
| **Drizzle** | 轻量级、SQL-like 类型安全 | 零依赖、接近原生 SQL |
| **Knex** | 查询构建器、灵活性优先 | 适合复杂查询、手动类型管理 |

3. 禁止在业务代码中直接拼接 SQL 字符串，必须使用 ORM 提供的参数化查询或查询构建器。
4. 所有数据库操作必须通过 repository 层封装，controller 和 service 禁止直接操作 ORM 客户端。

检查方式：架构评审 + 代码审查
阻断级别：阻断合并

---

## N+1 查询防护（MUST）

1. 列表查询必须评估是否存在 N+1 问题，关联数据必须使用 eager loading 或 join 查询。
2. Prisma 项目必须使用 `include` 或 `select` 显式指定关联加载，禁止在循环中逐条查询关联数据。
3. TypeORM 项目必须使用 `relations`、`leftJoinAndSelect` 或 `QueryBuilder` 进行关联查询。
4. 批量查询 ID 列表时必须使用 `WHERE IN` 批量查询，禁止循环单条查询。
5. CI 中推荐启用查询日志审查（开发环境开启 Prisma `log: ['query']` 或 TypeORM `logging: true`），定期排查慢查询和 N+1。

### SHOULD
1. 推荐使用 DataLoader 模式解决 GraphQL 场景下的 N+1 问题。
2. 推荐在开发环境使用查询分析工具（如 `prisma-query-inspector`）实时检测 N+1。

检查方式：查询日志分析 + 性能测试
阻断级别：阻断合并

---

## 事务管理（MUST）

1. 事务边界定义在 service 层，repository 层仅执行事务上下文内的数据操作。
2. Prisma 项目使用 `prisma.$transaction()` 进行事务操作，推荐使用交互式事务（`$transaction(async (tx) => { ... })`）。
3. TypeORM 项目使用 `DataSource.transaction()` 或 `QueryRunner` 管理事务。
4. 事务必须设置超时（推荐 30 秒），超时自动回滚，禁止长事务（> 60 秒）。
5. 事务内禁止执行外部 API 调用、消息发送等不可回滚操作，必须使用"先写库后异步处理"模式。
6. 事务回滚后必须记录日志，包含操作上下文和回滚原因。
7. 嵌套事务必须使用 Savepoint 机制，禁止手动管理嵌套事务状态。

### SHOULD
1. 推荐使用 Unit of Work 模式管理复杂事务。
2. 推荐将事务方法标记为 `@Transactional()`（通过自定义装饰器实现）。

检查方式：集成测试（含回滚场景）
阻断级别：阻断合并

---

## 数据库迁移（MUST）

1. 数据库结构变更必须通过迁移文件管理，禁止手动修改生产数据库结构。
2. Prisma 项目使用 `prisma migrate dev`（开发）和 `prisma migrate deploy`（生产）。
3. TypeORM 项目使用 `migration:generate` 和 `migration:run`。
4. 迁移文件必须纳入版本控制，文件名包含时间戳（自动生成）。
5. 迁移必须支持回滚（down migration），生产环境执行迁移前必须在预发布环境验证。
6. 破坏性迁移（删除列、修改类型、删除表）必须分阶段执行：
   - 第一阶段：新增列/表，双写。
   - 第二阶段：迁移数据，切换读取。
   - 第三阶段：清理旧结构。
7. 迁移执行必须在事务中，失败自动回滚，禁止部分执行的迁移。

### SHOULD
1. 推荐在 CI 中自动执行迁移到测试数据库并验证 Schema 一致性。
2. 推荐大表变更使用 `pt-online-schema-change` 或 `gh-ost` 避免锁表。

检查方式：CI 迁移验证 + 评审
阻断级别：阻断合并

---

## 查询优化（MUST）

1. 查询条件涉及的列必须有合适的索引，新增查询必须评估索引需求。
2. 禁止使用 `SELECT *`（Prisma `findMany()` 无 `select`/`include`），必须明确指定所需字段。
3. 分页查询必须限制 `pageSize`（推荐最大 100），禁止一次查询返回无限制数据。
4. 批量写入推荐使用 `createMany` / `insertMany`，禁止循环逐条插入。
5. 慢查询（> 200ms）必须记录日志并定期优化。
6. 统计查询推荐使用数据库原生聚合函数，禁止将全量数据拉到应用层进行聚合。

### SHOULD
1. 推荐为常用复合查询创建复合索引，注意索引字段顺序。
2. 推荐定期执行 `EXPLAIN ANALYZE` 审查关键查询的执行计划。
3. 推荐读写分离场景使用只读副本处理查询请求。

检查方式：慢查询日志 + 索引审查
阻断级别：告警记录

---

## 数据模型规范（MUST）

1. 每个 Entity 必须有主键，推荐使用自增 ID 或 UUID（禁止使用业务字段作为主键）。
2. 必须包含 `createdAt`、`updatedAt` 时间戳字段，使用数据库自动管理。
3. 软删除场景必须使用 `deletedAt` 字段（Prisma 通过中间件实现，TypeORM 使用 `@DeleteDateColumn`）。
4. 枚举字段推荐使用数据库原生 ENUM 或字符串 + 应用层校验，禁止使用魔法数字。
5. 金额字段必须使用 `Decimal` 类型（Prisma `Decimal`、TypeORM `decimal`），禁止使用 `float`/`double`。

---

## 数据库操作日志（MUST）
1. 所有数据库操作必须可追踪，通过 ORM 的日志机制记录操作日志，至少包含：操作类型、目标表、执行耗时、是否成功。
2. 慢查询阈值必须可配置（建议默认 200ms），超过阈值的查询必须以 `warn` 级别记录日志并标记 `slowQuery: true`。
3. 慢查询日志必须与正常应用日志输出到同一日志通道（不单独分离文件），通过日志级别和标记字段区分。
4. 慢查询日志须包含：SQL 语句（脱敏后）、执行耗时、影响行数、调用来源。
5. Prisma 项目推荐使用 `$on('query')` 事件监听；TypeORM 项目推荐使用自定义 Logger 实现。
6. 开发环境允许开启全量查询日志（Prisma `log: ['query']`、TypeORM `logging: true`），生产环境仅记录慢查询和错误查询。

检查方式：慢查询日志审查 + 代码审查
阻断级别：阻断合并
