# Python 服务端场景实践(按需加载)

硬约束与红线见 `baseline.md`。默认技术栈 FastAPI + SQLAlchemy 2.0 + Alembic;Django/Flask 项目按框架惯例对应执行。

## 项目结构(FastAPI 模块化单体)

```text
app/
├── main.py                 # 创建 FastAPI 实例、注册路由
├── lifespan.py             # 组件初始化/关闭(lifespan 上下文)
├── settings.py             # pydantic-settings 配置类
├── core/                   # exceptions、response、security、middleware(cors/request_id/logging/error_handler)
├── platform/               # database、redis_client、object_storage、jwt_handler
├── cache/keys.py           # 缓存键集中定义
├── shared/                 # 跨模块技术组件(无业务语义)
└── modules/<module>/       # router、schemas、service、repository、models、exceptions、dependencies、tasks
alembic/                    # 迁移(env.py + versions/)
configs/                    # .env.example、gunicorn.conf.py
tests/                      # conftest.py + unit/ + integration/
```

1. admin 与 user 等作用域独立 router + 独立 `dependencies.py` 鉴权,禁止 if 分支混合。
2. 模块间禁止直接调用对方 repository;跨模块走 service 接口。

## API 契约

1. FastAPI 自动生成 OpenAPI 即契约源(Code-First):response_model 必须显式声明,schema 字段带描述与示例。
2. API 变更同 PR 确认生成的 OpenAPI 无非预期漂移;破坏性变更评审升版本。
3. Django/Flask 项目用 drf-spectacular / flask-smorest 等生成 OpenAPI,保持单一契约源。

## 组件初始化与生命周期

1. FastAPI 用 `lifespan` 上下文统一初始化/关闭组件(顺序:config → logger → db engine → redis → storage;关闭反向)。
2. 组件经 `Depends` 注入;`AsyncEngine` 全局单例(启动时创建)但 `AsyncSession` 每请求创建。
3. 提供 `/healthz`(存活)与 `/readyz`(就绪,检查 DB/Redis 关键依赖)。
4. 部署:uvicorn(容器内单进程多副本)或 gunicorn + uvicorn worker;worker 数与超时走配置;优雅停机依赖服务器信号处理,长任务响应取消。

## 数据库访问(SQLAlchemy 2.0)

1. 声明式映射用 `DeclarativeBase` + `Mapped[T]`/`mapped_column` 类型注解风格;禁止 legacy Query API。
2. 查询用 `select()` 显式列或 `load_only()`,禁止取整实体只用部分字段;批量写入单批 ≤ 1000。
3. N+1:关联加载必须 `selectinload()`/`joinedload()` 预加载;异步 Session 触发延迟加载会抛 `MissingGreenlet`,必须 eager loading。
4. 事务:service 层 `async with session.begin():` 管理;事务内禁止远程调用;Django 用 `transaction.atomic()`。
5. 连接池走配置:`pool_size`、`max_overflow`、`pool_timeout`、`pool_recycle`(短于 DB wait_timeout)、`pool_pre_ping=True`。
6. 迁移:Alembic(Django 用内置 migrate);每次迁移必须可回滚(`downgrade()`);禁止修改历史迁移。
7. 慢查询:SQLAlchemy 事件钩子记录,阈值可配置(默认 200ms),WARNING 级 + 脱敏 SQL;生产禁开 `echo=True`。

## 缓存

1. 分布式用 Redis(`redis.asyncio`);键集中定义 `cache/keys.py`,格式 `{服务}:{业务域}:{标识}`;全部设 TTL 加随机偏移。
2. Cache-Aside:先写库后删缓存,删除失败有补偿;穿透用空值缓存(30–60s)/布隆过滤器;击穿热点键用分布式锁或逻辑过期;强一致数据禁止以缓存为数据源。
3. Redis 分布式锁设超时 + 唯一标识防误释放。

## 后台任务

1. 异步任务统一选型:Celery(重量级、需要重试/编排)或 arq/Dramatiq(轻量异步),项目内唯一。
2. 定时任务多实例必须分布式锁(Redis),锁标识含任务名+周期;任务幂等,执行结果持久化。
3. 失败告警 + 重试 ≤ 3 次带退避;禁止无限重试;长任务设超时。
4. FastAPI `BackgroundTasks` 仅用于请求后轻量操作,不承载可靠任务(进程重启即丢失)。

## 文件存储

1. 统一对象存储(MinIO/OSS/S3),客户端封装在 `platform/object_storage.py`,业务经接口调用;禁止存本地磁盘。
2. 上传校验大小 + MIME/文件头白名单;存储名 UUID 重命名;大文件分片/流式,禁止整读内存;元信息入库。
3. 访问分级:公开(CDN)/受限(签名 URL ≤ 30min)/敏感(SSE + 短签名 + 审计),不同级别不同 Bucket。

## 可观测性

1. OpenTelemetry SDK(`opentelemetry-instrumentation-fastapi/sqlalchemy/redis` 自动埋点),W3C Trace Context 透传。
2. 指标:RED + 下游依赖成功率/耗时 + 进程指标;Prometheus 格式暴露于管理端口。
3. 告警必配:错误率 >1%、P99 >1s、实例不可用、连接池耗尽、任务队列积压;分级 P0/P1/P2 并指定负责人。

## 性能

1. 响应时间目标 P95 ≤ 200ms、P99 ≤ 500ms;耗时操作(导出、批量导入)异步化返回任务 ID。
2. JSON 序列化热路径可用 `orjson`(FastAPI 配 `ORJSONResponse`);大数据集流式响应(`StreamingResponse`)。
3. CPU 密集任务放独立 worker 进程,不阻塞事件循环;性能排查用 `py-spy`/`cProfile`。

## 测试

1. pytest + pytest-asyncio;service 层单元测试 mock repository;repository 集成测试用 testcontainers 起真实数据库;API 层用 `httpx.AsyncClient` + `ASGITransport`。
2. 表驱动参数化(`@pytest.mark.parametrize`)覆盖正常/边界/异常;修复缺陷补回归测试。
3. Fixture 集中在 `conftest.py`;测试数据用 factory(factory_boy)而非手写字典。

## 微服务差异(microservice profile)

1. 每服务独立仓库或 monorepo 明确边界;禁止共享 ORM model,服务间只通过契约(OpenAPI/proto)通信。
2. 同步调用统一 httpx / gRPC(grpclib、grpcio)二选一;跨服务一致性用 Saga/Outbox,禁止分布式大事务。
3. 弹性:下游调用配置超时、重试(tenacity)、熔断;超时逐层递减。
4. 消息:统一 broker 选型(RabbitMQ/Kafka/Redis Streams);消费幂等 + 死信 + 积压告警;消息头透传 trace 上下文。
5. 容器:多阶段构建(uv 安装依赖层缓存)、非 root 运行、`python:3.12-slim` 基础镜像;K8s 配探针与资源限制;发布走灰度。
