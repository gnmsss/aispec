# rules/go-server/common/database-access.md

## 数据库类型约束（MUST）
1. 服务必须明确数据库类型，仅允许 `mysql` 或 `postgresql` 两种选择。
2. 数据库类型必须与配置文件中的 `database.type` 一致，不得在代码中隐式推断。
3. 若需求中出现数据库接入但未指定类型，必须先反馈并由用户选择后再继续实现。

## ORM / 查询构建器选型（MUST）
1. 项目必须选定唯一的数据库访问方案，禁止混用多套 ORM/查询构建器。
2. 推荐方案：**GORM**（全功能 ORM）或 **sqlx**（轻量映射 + 原生 SQL）。
3. 选型后必须在文档中记录，变更需经架构评审。
4. 禁止在业务代码中直接使用 `database/sql` 裸接口（除非项目统一选型为 sqlx）。

| 方案 | 适用场景 | 特点 |
|------|---------|------|
| **GORM** | 通用 CRUD、模型驱动开发 | 丰富的关联、Hook、迁移支持 |
| **sqlx** | 对 SQL 控制要求高、性能敏感 | 轻量映射、原生 SQL、学习成本低 |
| **Ent** | 图模型、复杂关联查询 | 代码生成、类型安全、强 Schema 约束 |

## 连接池管理（MUST）
1. 必须通过 `sql.DB` 的连接池参数控制资源使用，禁止使用默认无限连接。
2. 必须配置以下参数（通过配置文件可调）：
   - `SetMaxOpenConns`：最大打开连接数（建议根据服务实例数和数据库上限计算）。
   - `SetMaxIdleConns`：最大空闲连接数（建议 ≤ MaxOpenConns 的 50%）。
   - `SetConnMaxLifetime`：连接最大存活时间（建议 5-10 分钟，必须短于数据库 `wait_timeout`）。
   - `SetConnMaxIdleTime`：空闲连接最大存活时间（建议 1-3 分钟）。
3. 服务启动时必须执行 `db.Ping()` 或等效健康检查，确认数据库可达。
4. 服务优雅停机时必须调用 `db.Close()` 释放所有连接。

### 连接池配置示例
```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(50)
sqlDB.SetMaxIdleConns(25)
sqlDB.SetConnMaxLifetime(5 * time.Minute)
sqlDB.SetConnMaxIdleTime(1 * time.Minute)
```

## 数据模型约束（MUST）
1. 必须建立持久化模型（Persistent Model）映射数据库结构，作为常规查询与写入的默认模型。
2. 常规查询（单表或稳定业务查询）必须使用持久化模型，禁止长期使用临时结构体或 `map` 直接承载结果。
3. 分析统计、报表、多表联合聚合等场景可使用临时读模型（Read Model / Query Model）替代持久化模型。
4. 临时读模型仅用于查询结果承载，不得直接用于写入、更新或替代持久化模型的领域语义。
5. 新增临时读模型必须标注用途与作用域，并按职责独立文件命名（如 `order_stat_row.go`、`user_report_item.go`）。

## 查询与写入规范（MUST）
1. SQL 必须显式列字段，禁止 `SELECT *`。
2. 参数化查询是强制要求，禁止字符串拼接 SQL。
3. 批量写入和批量更新必须限制单批大小（建议 ≤ 500 条），避免锁和日志放大。
4. 更新和删除操作必须带 `WHERE` 条件，GORM 的全局更新/删除保护（`gorm.Config{AllowGlobalUpdate: false}`）必须保持启用。
5. 复杂查询推荐使用 `Scopes` 链式组合，避免在业务层硬编码 SQL 片段。

## N+1 查询防护（MUST）
1. 关联数据加载必须使用 `Preload` / `Joins`（GORM）或手动批量查询，禁止在循环中逐条查询。
2. 列表接口的关联查询必须在 Code Review 中检查是否存在 N+1 问题。
3. 推荐在开发环境启用 SQL 日志全量输出，通过日志条数快速发现 N+1 模式。

## 事务边界（MUST）
1. 事务边界在 `service` 层定义，`repository` 只执行事务上下文内操作。
2. 单次事务应短小，禁止在事务中执行远程调用或不确定时延逻辑。
3. 必须明确隔离级别与锁策略，避免隐式锁冲突。
4. GORM 事务推荐使用 `db.Transaction(func(tx *gorm.DB) error { ... })`，自动处理 Commit/Rollback。
5. 嵌套事务使用 `SavePoint`，禁止在事务中再次调用 `db.Begin()`。

### 事务封装示例
```go
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderReq) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        order := mapToOrder(req)
        if err := tx.Create(&order).Error; err != nil {
            return fmt.Errorf("repo.CreateOrder: %w", err)
        }
        if err := tx.Create(&orderItems).Error; err != nil {
            return fmt.Errorf("repo.CreateOrderItems: %w", err)
        }
        return nil
    })
}
```

## Schema 变更（MUST）
1. 数据库结构或初始化数据变更必须遵守 `rules/database/database.md`。
2. 禁止修改历史迁移脚本，变更必须通过新增脚本交付。
3. 禁止在应用启动时自动执行 `AutoMigrate`（生产环境），迁移必须独立执行。

## 数据库操作日志（MUST）
1. 所有数据库操作必须通过 ORM/驱动的日志钩子记录操作日志，至少包含：操作类型（SELECT/INSERT/UPDATE/DELETE）、目标表、执行耗时、是否成功。
2. 慢查询阈值必须可配置（建议默认 200ms），超过阈值的查询必须以 `WARN` 级别记录日志并标记 `slow_query: true`。
3. 慢查询日志必须与正常应用日志输出到同一日志通道（不单独分离文件），通过日志级别和标记字段区分。
4. 慢查询日志须包含：SQL 语句（脱敏后）、执行耗时、影响行数、调用来源（调用方函数或模块）。
5. 推荐使用 GORM 的 `Logger` 接口自定义实现，将慢查询与操作日志接入项目统一的结构化日志组件。
6. 开发环境允许开启全量 SQL 日志（DEBUG 级别），生产环境仅记录慢查询和错误查询。

检查方式：慢查询日志审查 + EXPLAIN 审查 + 代码审查
阻断级别：阻断合并
