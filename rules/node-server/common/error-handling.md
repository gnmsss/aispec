# rules/node-server/common/error-handling.md

## 异常分类体系

### MUST
1. 异常必须按以下两大类区分：
   - **业务异常（BusinessException）**：用户可感知的业务错误（参数校验失败、资源不存在、权限不足等），携带业务错误码和用户可读信息。
   - **系统异常（SystemException）**：基础设施故障（数据库连接失败、Redis 超时、第三方服务不可用等），记录日志后返回通用错误提示。
2. 业务异常必须继承统一基类 `BusinessException`，携带 `errorCode`（业务错误码）、`message`（用户提示）、`httpStatus`（HTTP 状态码）。
3. 系统异常禁止将原始错误信息返回给客户端（如数据库错误、堆栈信息），必须映射为通用的 "服务器内部错误" 提示。
4. 异常类必须按业务域组织，如 `UserBusinessException`、`OrderBusinessException`，禁止所有业务错误使用同一个异常类。

### SHOULD
1. 推荐定义异常层级：`BaseException` → `BusinessException` / `SystemException` → `UserNotFoundException` / `DuplicateOrderException` 等具体异常。
2. 推荐异常类包含 `cause` 属性，保留错误链便于排查。

---

## NestJS 异常处理（MUST）

1. 必须实现全局 `ExceptionFilter`，统一捕获所有异常并映射为标准响应结构。
2. `ExceptionFilter` 必须区分 `BusinessException`、`HttpException`、`ValidationError`（class-validator）和未知异常，分别处理：
   - `BusinessException`：返回业务错误码和用户提示。
   - `HttpException`：返回对应 HTTP 状态码和错误信息。
   - `ValidationError`：返回 `400` + 字段级校验错误详情。
   - 未知异常：返回 `500` + 通用错误提示，记录完整错误堆栈到日志。
3. `ExceptionFilter` 必须在每次错误响应中注入 `requestId` 和 `timestamp`。
4. `ExceptionFilter` 必须注册为全局 Filter（`APP_FILTER`），禁止在每个 controller 中分别注册。
5. controller 中禁止使用 `try-catch` 捕获业务异常后手动构造错误响应，必须抛出异常交由全局 Filter 处理。

---

## Express/Fastify 错误中间件（MUST）

1. Express 项目必须在路由注册之后添加全局错误中间件（四参数 `(err, req, res, next)`）。
2. Fastify 项目必须使用 `setErrorHandler` 注册全局错误处理器。
3. 错误中间件/处理器必须与 NestJS ExceptionFilter 保持相同的分类逻辑和响应结构。
4. 禁止在路由 handler 中 `res.status(500).json({...})` 手动构造错误响应，必须 `next(err)` 或 `throw` 交由全局处理。

---

## 错误码规范（MUST）

1. 业务错误码必须遵循统一编码规则，推荐格式：`{模块}{子类}{序号}`，如 `USER_001`（用户模块第 1 号错误）。
2. 错误码必须集中注册（如 `error-codes.ts` 或每个模块的 `errors/` 目录），禁止在代码中散写魔法字符串。
3. 业务异常实例必须引用集中注册的错误码创建，禁止 `throw new BusinessException("随意编码", "随意消息")` 绕过错误码注册表。
4. 每个错误码必须有唯一性，禁止不同错误复用同一错误码。
5. 错误码文档必须包含：错误码、HTTP 状态码映射、中文描述、可能的触发场景和建议处理方式。
6. 新增错误码必须经过 PR 评审，禁止私自添加未记录的错误码。

### SHOULD
1. 推荐按模块拆分错误码文件：`user.errors.ts`、`order.errors.ts` 等。
2. 推荐错误码同时支持 i18n，错误消息作为 key 映射到不同语言。

检查方式：错误码注册表审查 + 集成测试验证
阻断级别：阻断合并

---

## 异常传播规则（MUST）

1. `repository` 层捕获数据访问错误后，必须包装为语义明确的异常重新抛出（如 `EntityNotFoundException`），禁止直接透传 ORM 原始错误。
2. `service` 层可直接抛出 `BusinessException`，交由上层统一处理。
3. `controller` 层禁止吞掉异常（空 `catch` 块），禁止手动 `try-catch` 后返回自定义错误格式。
4. 异步操作（Promise）的错误必须被正确处理，禁止 unhandled rejection；NestJS 中使用 `async/await` 确保异常冒泡。
5. 进程级必须注册 `unhandledRejection` 和 `uncaughtException` 监听器，记录错误日志后执行优雅退出。

### SHOULD
1. 推荐 `service` 层在捕获下游异常时使用 `cause` 保留错误链：`throw new BusinessException('...', { cause: originalError })`。
2. 推荐在错误日志中包含请求上下文（userId、requestId、路由路径）。

---

## 错误监控与告警（SHOULD）

1. 5xx 错误数量/比率纳入监控指标，超阈值触发告警。
2. 关键业务异常（支付失败、权限拒绝等）纳入独立告警通道。
3. 错误日志必须包含足够的排查信息：`requestId`、`userId`、`method`、`path`、`errorCode`、`stack`。
4. 推荐集成 Sentry 等错误追踪平台，自动聚合和通知。
