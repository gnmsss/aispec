# Python 服务端基线(必载)

适用:所有 Python 服务端代码(FastAPI / Django / Flask,含 API、后台任务、Worker)。
冲突优先级:`rules/database.md` > `rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Python 3.12+(以 `pyproject.toml` / `.python-version` 为准),禁止 EOL 版本;升级版本单独提交。
2. 包管理统一使用 `uv`(存量项目可保留 poetry,同一项目禁止混用);lockfile(`uv.lock`/`poetry.lock`)必须入库。
3. 工具链:`ruff`(lint + format)+ `mypy --strict` 或 `pyright`,配置入库(`pyproject.toml`),CI 阻断;提交前 `ruff check` 与类型检查无新增错误。
4. CI 集成依赖漏洞扫描(`pip-audit`/`osv-scanner`),高危漏洞(CVSS ≥ 7.0)阻断合并;商用项目禁止 GPL/AGPL 依赖;依赖更新单独提交。
5. 类型注解全覆盖:公开函数必须有参数与返回类型;使用内置泛型语法(`list[X]`、`X | None`),禁止 `typing.List`/`typing.Optional`(3.10+ 项目)。
6. Pydantic v2 强制;禁止 v1 API(`class Config:`、`@validator`)。

## 分层架构(MUST)

1. 依赖单向:`router(transport) -> service -> repository`;入口(`main.py`/`lifespan.py`)只做组装与生命周期。
2. router 只做协议适配,禁止直接操作数据库;service 承载用例编排与事务边界;repository 只做数据访问。
3. ORM model 禁止直接作为 API 响应返回,必须经 Pydantic schema 转换;协议层 schema 不下沉到 repository。
4. 通用无业务语义能力放 `core/`;业务模块按 `modules/<module>/{router,schemas,service,repository,models,exceptions}` 组织,一文件一责任。

## 错误处理(MUST)

1. 禁止裸 `except:`;禁止 `except Exception: pass` 吞异常,必须记日志或重抛。
2. 业务异常继承统一基类并携带结构化错误码,按模块集中定义(`modules/<module>/exceptions.py`)。
3. 统一异常处理器(FastAPI `exception_handler` / Django middleware)完成业务异常 → 错误码映射,系统异常记完整日志后返回固定文案;禁止散落式 try/except 映射。
4. API 响应禁止包含原始堆栈、SQL、内部地址、密钥。

## 异步(MUST)

1. 异步框架(FastAPI)中禁止调用同步阻塞函数(`time.sleep`、`requests`、同步文件 I/O);HTTP 客户端用 `httpx.AsyncClient`,阻塞操作走 `run_in_executor` 或改异步库。
2. 异步项目数据库必须用 SQLAlchemy `AsyncSession`,禁止异步上下文用同步 Session。
3. 客户端(httpx/Redis)全局复用并配置超时,禁止每请求新建。

## API 响应(MUST)

1. 统一响应结构:`code`、`message`、`data`、`request_id`、`timestamp`;`request_id` 中间件注入。
2. 语义化状态码:2xx/4xx/5xx,禁止失败统一 200;校验错误 422/400 + 字段级 details;鉴权错误 401/403。
3. 对外时间统一 UTC;API 版本化(`/api/v1`);列表接口必须分页并限制 page_size 上限;写接口定义幂等策略。

## 配置(MUST)

1. 统一 `pydantic-settings`:配置类继承 `BaseSettings`,类型注解 + 校验,启动即验证失败退出;禁止业务代码直接 `os.getenv()`。
2. `.env` 入 `.gitignore`,仅 `.env.example` 入库;密钥来自环境变量或密钥管理服务。
3. Profile 显式指定并白名单校验(`dev/test/staging/prod`),启动日志输出当前 profile;生产禁止 `DEBUG=True`。
4. 数据库显式声明 `type`(仅 `mysql`/`postgresql`),未指定先询问;外部依赖参数(超时、连接池)全部走配置。
5. CORS 白名单走配置;`allow_credentials=True` 时禁止 `allow_origins=["*"]`。

## 日志(MUST)

1. 结构化日志(`structlog` 或 logging + JSON formatter 统一封装);禁止 `print()`/`pprint()` 输出日志。
2. 禁止提交 `breakpoint()`/`pdb`/`ipdb` 调试代码;每条请求链路记录 `request_id`/`trace_id`。
3. 禁止输出密码、令牌、证件号等敏感信息。

## 红线(MUST NOT)

1. 禁止字符串拼接 SQL(f-string 拼 SQL 同罪),必须参数化。
2. 禁止 `import *`(`__init__.py` 显式重导出除外);禁止可变默认参数(`def f(x=[])`)。
3. 禁止生产代码使用 `eval()`/`exec()`/动态 `__import__()`。
4. 禁止模块级全局可变组件实例(全局 `db`、`redis_client`);组件经依赖注入(FastAPI `Depends`)获取;禁止在 router/service 直接构造客户端(`create_engine()`、`Redis()`、`boto3.client()`)。
5. 禁止应用启动时自动执行迁移(`alembic upgrade head`),迁移独立执行;结构变更遵循 `rules/database.md`。
6. 禁止无上限缓存/队列/无超时外部调用。
7. 测试禁止依赖外部真实服务(用 mock 或 testcontainers);禁止无断言测试。
