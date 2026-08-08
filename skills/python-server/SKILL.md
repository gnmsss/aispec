---
name: python-server
description: Python 服务端工程规范。编写或修改 Python 服务端代码(FastAPI/Django/Flask API、Celery 任务、Worker)、审查 Python 服务端代码变更、或初始化 Python 服务端项目时使用。
---

# Python 服务端规范

## 编码引导

1. 任何 Python 服务端编码任务,先读 `rules/python-server/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/python-server/practices.md` 对应章节:项目结构、API 契约、组件初始化、SQLAlchemy 数据访问、缓存、后台任务、文件存储、可观测性、性能、测试;微服务项目额外读文末「微服务差异」。
3. 跨域联动:
   - API 契约变更 / 前后端联调 → `rules/collaboration.md`
   - 数据库 schema / 迁移 → `rules/database.md`(优先级最高)
   - 认证、密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:router → service → repository 单向;router 未直接操作数据库
2. 类型:公开函数有完整类型注解;用内置泛型语法;Pydantic v2 API
3. 异常:无裸 except、无吞异常;业务异常带错误码走统一处理器;响应不泄露堆栈/SQL
4. 异步:异步上下文无同步阻塞调用(requests/time.sleep/同步 Session);客户端复用且有超时
5. SQL:无拼接 SQL;查询有投影;无 N+1(selectinload/joinedload);批量有上限
6. 事务:边界在 service,`session.begin()` 管理,事务内无远程调用
7. 响应:统一包结构,语义化状态码,无 200 兜底;ORM model 未直接返回
8. 配置:pydantic-settings 统一管理,无 os.getenv 散调,无硬编码密钥;生产无 DEBUG=True
9. 日志:结构化,无 print/断点调试残留,无敏感信息
10. 迁移:未修改历史迁移;未在启动时自动执行迁移
11. 测试:有断言;不依赖外部真实服务;缺陷修复有回归测试
12. 依赖:lockfile 已更新入库;新依赖许可证合规

## 项目脚手架

初始化新 Python 服务端项目时:

1. 确认(未指定必须先询问):框架(FastAPI/Django)、数据库类型(mysql/postgresql)、是否需要异步任务队列。
2. Python 3.12+ + uv 初始化(`uv init` + `pyproject.toml`);配置 ruff(lint+format)+ mypy strict,lockfile 入库。
3. 按 `rules/python-server/practices.md`「项目结构」建 `app/{core,platform,modules}` + `alembic/` + `tests/`。
4. 落地骨架:pydantic-settings 配置类、统一响应结构、全局异常处理器、request_id 中间件、结构化日志、`/healthz` `/readyz`、lifespan 组件管理。
5. 首个业务模块按 `modules/<module>/{router,schemas,service,repository,models,exceptions}` 打样,配 pytest + conftest。
