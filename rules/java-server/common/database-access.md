# rules/java-server/common/database-access.md

## 数据库类型约束

### MUST
1. 服务必须明确数据库类型，仅允许 MySQL 或 PostgreSQL 两种选择。
2. 数据库类型必须与配置文件中的 `spring.datasource.driver-class-name` 一致，不得在代码中隐式推断。
3. 若需求中出现数据库接入但未指定类型，必须先确认数据库类型后再继续实现。

## ORM 框架选型（MUST）

1. 项目必须选定 JPA（Spring Data JPA + Hibernate）或 MyBatis/MyBatis-Plus（二选一），禁止同一项目混用。
2. 选型必须在项目初始化阶段确定并记录。
3. JPA 适用于领域模型驱动、CRUD 为主的场景；MyBatis 适用于复杂 SQL、报表查询、需要精细控制 SQL 的场景。

## JPA 规范（MUST，选用 JPA 时适用）

1. Entity 类必须标注 `@Entity`，表名使用 `@Table(name = "xxx")` 显式声明。
2. 主键策略必须显式声明（`@GeneratedValue(strategy = ...)`），禁止依赖默认策略。
3. 禁止在 Entity 上使用 `@Data`（Lombok），必须手写 `equals`/`hashCode`（基于业务主键或数据库主键）。
4. 关联映射（`@OneToMany`、`@ManyToOne`）默认使用懒加载（`fetch = FetchType.LAZY`），禁止使用 `FetchType.EAGER`。
5. N+1 查询必须消除：使用 `@EntityGraph`、`JOIN FETCH` 或 `@BatchSize` 批量加载关联。
6. 禁止使用 `CascadeType.ALL` 或 `CascadeType.REMOVE` 做级联删除，必须在 Service 层显式管理。
7. 自定义查询使用 `@Query` + JPQL 或原生 SQL，查询参数必须使用命名参数（`:paramName`）。

## MyBatis 规范（MUST，选用 MyBatis 时适用）

1. SQL 映射文件（`*Mapper.xml`）必须与 Mapper 接口同包或在 `resources/mapper/` 下统一管理。
2. SQL 必须显式列字段，禁止 `SELECT *`。
3. 动态 SQL 使用 `<if>`、`<choose>`、`<foreach>` 标签，禁止 Java 代码拼接 SQL 字符串。
4. 参数化查询是强制要求，使用 `#{}` 占位符，禁止使用 `${}` 做值拼接（仅允许在表名/列名等确定安全的动态场景使用 `${}`，需代码审查确认）。
5. 复杂查询推荐使用 MyBatis-Plus 的 `Wrapper` API，简单查询使用 Mapper 接口方法。
6. 分页查询必须使用分页插件（PageHelper 或 MyBatis-Plus 分页），禁止手写 `LIMIT OFFSET`。

## 数据模型约束（MUST）

1. 必须建立持久化模型（Entity / Model）映射数据库结构，作为常规查询与写入的默认模型。
2. 常规查询必须使用持久化模型，禁止长期使用 `Map<String, Object>` 或临时 POJO 承载结果。
3. 分析统计、报表、多表联合聚合等场景可使用 DTO/VO 替代持久化模型。
4. DTO/VO 仅用于查询结果承载，不得直接用于写入或替代持久化模型。
5. 新增 DTO/VO 必须标注用途，按职责独立文件命名。

## 查询与写入规范（MUST）

1. SQL 必须显式列字段，禁止 `SELECT *`。
2. 参数化查询是强制要求，禁止字符串拼接 SQL（防 SQL 注入）。
3. 批量写入和批量更新必须限制单批大小（建议 ≤ 500），避免锁和日志放大。
4. 列表查询必须分页，禁止无 `LIMIT` 的 `SELECT`（管理脚本或迁移任务除外）。
5. `WHERE` 条件中参与过滤的字段必须有索引覆盖；新增查询必须附 `EXPLAIN` 结果或索引说明。
6. N+1 查询必须消除：批量场景使用 `WHERE IN (...)` 或 `JOIN` 替代循环单条查询。

## 事务管理（MUST）

1. 事务边界在 `Service` 层定义，使用 `@Transactional` 注解管理。
2. `@Transactional` 必须显式指定 `rollbackFor = Exception.class`（或具体异常），避免 checked exception 不回滚的陷阱。
3. 只读查询使用 `@Transactional(readOnly = true)`，提升数据库性能。
4. 单次事务应短小，禁止在事务中执行远程调用（HTTP/RPC）或不确定时延逻辑。
5. `@Transactional` 禁止标注在 `private` 方法上（Spring AOP 代理限制，不生效）。
6. 同一类内部方法调用不会触发事务代理，需跨 Bean 调用或使用 `TransactionTemplate` 编程式事务。
7. 必须明确隔离级别与锁策略，默认使用数据库默认隔离级别，变更需注释说明。

## Schema 变更（MUST）

1. 数据库结构变更必须通过 Flyway（推荐）或 Liquibase 管理迁移脚本。
2. 迁移脚本必须纳入版本控制，与业务代码同仓管理。
3. 禁止修改历史迁移脚本，变更必须通过新增脚本交付。
4. 迁移脚本文件名遵循版本号规范（如 `V1.0.1__add_order_status_column.sql`）。
5. 生产环境迁移必须经过评审，高风险变更（大表 DDL、数据迁移）须有回滚方案。

## 数据库操作日志（MUST）
1. 所有数据库操作必须可追踪，通过 ORM/连接池的日志机制记录操作日志，至少包含：操作类型、目标表、执行耗时、是否成功。
2. 慢查询阈值必须可配置（建议默认 200ms），超过阈值的查询必须以 `WARN` 级别记录日志并标记 `slow_query: true`。
3. 慢查询日志必须与正常应用日志输出到同一日志通道（不单独分离文件），通过日志级别和标记字段区分。
4. 慢查询日志须包含：SQL 语句（脱敏后）、执行耗时、影响行数、调用来源。
5. JPA 项目推荐通过 Hibernate 的 `hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS` 配置慢查询阈值；MyBatis 项目推荐通过拦截器实现。
6. 开发环境允许开启全量 SQL 日志（`spring.jpa.show-sql=true` 或 MyBatis `log-impl`），生产环境仅记录慢查询和错误查询。

检查方式：慢查询日志审查 + EXPLAIN 审查 + 代码审查
阻断级别：阻断合并
