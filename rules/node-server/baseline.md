# Node.js 服务端基线(必载)

适用:所有 Node.js 服务端代码(NestJS / Express / Fastify,含 API、后台任务、Worker)。
冲突优先级:`rules/database.md` > `rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Node.js 22+ LTS(推荐 24 LTS),`package.json` 的 `engines.node` 声明最低版本,CI 校验运行时一致;仓库放 `.nvmrc` 或 volta 配置。
2. TypeScript `strict: true`,禁止关闭任何 strict 子选项;新项目 ESM 优先(`"type": "module"`),禁止新代码使用 `require()`(动态加载与 CJS 互操作除外)。
3. 包管理统一 `pnpm`,`pnpm-lock.yaml` 入库;CI 用 `pnpm install --frozen-lockfile`。
4. ESLint(flat config)+ Prettier;启用 `typescript-eslint` recommended + `no-explicit-any`、`no-floating-promises`;husky + lint-staged 在 pre-commit 执行。
5. CI 集成 `pnpm audit`,高危漏洞(CVSS ≥ 7.0)阻断;商用项目禁止 GPL/AGPL 依赖;`devDependencies` 禁止进入生产代码;依赖更新单独提交。
6. 路径别名(`@/`)替代超过三层的相对导入;`scripts` 必含 `dev/build/start/lint/test`。

## 类型(MUST)

1. 禁止 `any`(用 `unknown` + 类型收窄或泛型);确需绕过用 `@ts-expect-error` 加注释,禁止 `@ts-ignore` 与 `as any`。
2. 公开函数与导出 API 必须有显式类型;DTO 校验用 zod 或 class-validator,边界处完成解析。

## 分层架构(MUST)

1. 依赖单向:`controller → service → repository`;`main.ts` 只做组装与生命周期。
2. controller 禁止直接操作数据库/缓存/存储/MQ,禁止直接引用 repository;service 禁止依赖 HTTP 框架类型(`Request`/`Response`),禁止直接写 ORM 查询;repository 只做数据访问。
3. ORM 实体/模型禁止直接作为 API 响应,必须经响应 DTO 转换;业务实体禁止放共享目录跨服务复用。

## 异步(MUST)

1. 统一 async/await,禁止回调嵌套;禁止 unhandled rejection(floating promise 由 lint 阻断)。
2. 请求处理路径禁止同步 I/O(`fs.*Sync`,启动加载配置除外);CPU 密集操作放 Worker Threads 或独立进程,禁止阻塞事件循环。
3. 禁止用 `setTimeout`/`setInterval` 实现定时任务或延迟队列(进程重启即丢失),必须用任务调度组件。

## 错误处理(MUST)

1. 业务异常统一基类 + 结构化错误码(集中定义 `error-codes.ts`);统一异常处理(NestJS `ExceptionFilter` / 框架错误中间件),禁止各 controller 散落映射。
2. 禁止空 `catch` 吞异常,每个 catch 必须记日志或重抛;系统错误(ORM 原始错误、堆栈)禁止原样返回。
3. 统一响应结构:`code`、`message`、`data`、`request_id`、`timestamp`;语义化状态码,禁止失败统一 200;校验错误 400 + 字段级 details。

## 配置(MUST)

1. 配置集中定义并做 Schema 校验(zod / NestJS `ConfigModule` + validation),启动即校验失败退出;禁止业务代码散读 `process.env`。
2. `.env` 入 `.gitignore`,仅 `.env.example` 入库;密钥来自环境变量或密钥管理服务,禁止硬编码 JWT 密钥/API Key。
3. 数据库显式声明类型(仅 `mysql`/`postgresql`),未指定先询问;CORS 白名单走配置,`credentials: true` 时禁止 `origin: '*'`。

## 日志(MUST)

1. 结构化日志(pino 或 NestJS Logger 统一封装);禁止 `console.*` 与 `debugger` 提交主分支。
2. 每条请求链路记录 `request_id`/`trace_id`;禁止输出密码、令牌、证件号等敏感信息。

## 红线(MUST NOT)

1. 禁止拼接用户输入到 SQL,必须参数化/查询构建器。
2. 禁止模块顶层建立数据库/Redis 连接;禁止全局变量暴露组件实例;禁止在 controller/service/Guard 中直接 `new PrismaClient()`/`new Redis()`,必须 DI 注入。
3. 禁止无上限缓存/队列/无超时外部调用;禁止无 `select`/`include` 约束的全字段查询;禁止循环内逐条查询(N+1)。
4. 禁止 `float`/`double` 存储金额,必须 `Decimal`。
5. 禁止手改生产数据库,结构变更走迁移文件并遵循 `rules/database.md`;禁止应用启动时自动执行迁移。
6. 禁止 MD5/SHA 直接哈希密码,必须 `bcrypt`/`argon2`;禁止生产启用 `--inspect` 调试端口。
7. 禁止未评审的破坏性接口变更。
