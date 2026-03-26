# rules/python-server/common/database-access.md

## 数据库类型约束
1. 服务必须明确数据库类型，仅允许 `mysql` 或 `postgresql` 两种选择。
2. 数据库类型必须与配置文件中的 `database.type` 一致，不得在代码中隐式推断。
3. 若需求中出现数据库接入但未指定类型，必须先反馈并由用户选择后再继续实现。

## ORM 选型
1. FastAPI/Flask 项目推荐使用 SQLAlchemy 2.0+，MUST 使用声明式映射（`DeclarativeBase`）和类型注解风格。
2. Django 项目使用 Django ORM，MUST 遵循 Django model 规范。
3. 异步项目 MUST 使用 SQLAlchemy 异步扩展（`AsyncSession`），禁止在异步上下文中使用同步 Session。
4. 禁止同一项目混用多种 ORM（SQLAlchemy + Django ORM），必须统一。

### SQLAlchemy 2.0 声明式映射示例
```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, DateTime
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(32), unique=True)
    email: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

## 数据模型约束
1. 必须建立持久化模型（ORM Model）映射数据库结构，作为常规查询与写入的默认模型。
2. 常规查询必须使用 ORM Model，禁止长期使用临时字典或裸 `Row` 对象承载结果。
3. 分析统计、报表、多表联合聚合等场景可使用专用查询 Schema（`TypedDict` 或 Pydantic Model）替代 ORM Model。
4. 临时查询 Schema 仅用于查询结果承载，不得直接用于写入。
5. 新增查询 Schema 必须标注用途与作用域，并按职责独立文件命名。

## 查询与写入规范
1. SQL 必须显式列字段，禁止 `SELECT *`（使用 ORM 的 `load_only()` 或显式 `select()`）。
2. 参数化查询是强制要求，禁止字符串拼接 SQL（使用 SQLAlchemy 绑定参数或 ORM 查询构建器）。
3. 批量写入和批量更新必须限制单批大小（建议 ≤ 1000），避免锁和日志放大。
4. 列表查询必须分页，禁止无 `LIMIT` 的 `SELECT`（管理脚本或迁移任务除外，需注释说明）。

## N+1 查询防护（MUST）
1. 关联查询 MUST 使用 `joinedload()` / `selectinload()` / `subqueryload()` 预加载，禁止循环中逐条查询。
2. 批量场景使用 `WHERE IN (...)` 替代循环单条查询。
3. 异步 Session 中访问延迟加载属性会触发 `MissingGreenlet` 错误，MUST 使用 eager loading 或显式查询。

### N+1 防护示例
```python
# 正确：使用 selectinload 预加载关联
stmt = select(User).options(selectinload(User.orders)).where(User.id.in_(user_ids))
result = await session.execute(stmt)

# 错误：循环中逐条查询
for user_id in user_ids:
    user = await session.get(User, user_id)  # N+1
    orders = await session.execute(select(Order).where(Order.user_id == user_id))
```

## 事务边界
1. 事务边界在 `service` 层定义，`repository` 只执行事务上下文内操作。
2. 单次事务应短小，禁止在事务中执行远程调用或不确定时延逻辑。
3. 必须明确隔离级别与锁策略，避免隐式锁冲突。
4. SQLAlchemy 异步 Session MUST 使用 `async with session.begin():` 管理事务，禁止手动 `commit()` 后遗漏 `rollback()`。
5. Django 项目 MUST 使用 `transaction.atomic()` 管理事务边界。

## 数据库迁移（Migration）
1. MUST 使用迁移工具管理 schema 变更：SQLAlchemy 项目使用 `Alembic`，Django 项目使用内置 `migrate`。
2. 禁止修改历史迁移脚本，变更必须通过新增迁移脚本交付。
3. 迁移脚本必须纳入版本控制，与业务代码同仓管理。
4. 每次迁移必须可回滚（提供 `downgrade()` 或 `RunSQL` 反向操作）。
5. 大表结构变更必须评估锁影响，必要时使用在线 DDL 工具（如 `pt-online-schema-change`、`gh-ost`）。

### Alembic 配置要求
```python
# alembic/env.py — 必须指向 SQLAlchemy Base.metadata
target_metadata = Base.metadata
```

## 数据库操作日志（MUST）
1. 所有数据库操作必须可追踪，通过 ORM 的日志机制或事件监听记录操作日志，至少包含：操作类型、目标表、执行耗时、是否成功。
2. 慢查询阈值必须可配置（建议默认 200ms），超过阈值的查询必须以 `WARNING` 级别记录日志并标记 `slow_query: True`。
3. 慢查询日志必须与正常应用日志输出到同一日志通道（不单独分离文件），通过日志级别和标记字段区分。
4. 慢查询日志须包含：SQL 语句（脱敏后）、执行耗时、影响行数、调用来源。
5. SQLAlchemy 项目推荐使用 `event.listen(engine, "before_cursor_execute")` / `"after_cursor_execute"` 事件对实现；Django 项目推荐使用 `django.db.connection.queries` 或中间件实现。
6. 开发环境允许开启全量 SQL 日志（SQLAlchemy `echo=True`），生产环境仅记录慢查询和错误查询。

检查方式：慢查询日志审查 + EXPLAIN 审查 + 代码审查
阻断级别：阻断合并
