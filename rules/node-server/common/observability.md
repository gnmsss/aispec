# rules/node-server/common/observability.md

## 结构化日志

### MUST
1. 必须使用结构化日志库（推荐 `pino`，备选 `winston`），禁止使用 `console.log` 输出日志。
2. 日志必须输出 JSON 格式（生产环境），包含以下标准字段：
   - `timestamp`：ISO 8601 格式时间戳
   - `level`：日志级别（`trace`/`debug`/`info`/`warn`/`error`/`fatal`）
   - `message`：人类可读的日志消息
   - `requestId`/`traceId`：请求追踪标识
   - `service`：服务名称
   - `context`：模块/类名称
3. NestJS 项目必须替换内建 Logger 为 `pino`（推荐使用 `nestjs-pino`），确保框架级日志也走统一通道。
4. 日志级别必须可通过环境变量配置（如 `LOG_LEVEL=info`），生产环境最低 `info`，开发环境可设 `debug`。
5. 日志内容必须脱敏：禁止打印密码、JWT 令牌、信用卡号、身份证号等敏感信息。
6. 请求/响应日志必须包含 `method`、`path`、`statusCode`、`duration`（ms），排除健康检查路径。
7. 错误日志必须包含完整的堆栈信息（`stack`）和错误上下文。

### SHOULD
1. 推荐使用 `pino-pretty` 美化开发环境日志输出，生产环境保持 JSON 格式。
2. 推荐为每个请求生成 `requestId`（UUID v4），通过 AsyncLocalStorage 在整个请求链路中传递。
3. 推荐将日志采集到 ELK/Loki 等集中式日志系统。

### 日志文件轮转（MUST，输出到文件时适用）
1. 当日志输出目标为文件时，必须配置日志轮转策略，禁止单文件无限增长。
2. 单个日志文件最大大小必须可配置（建议默认 100MB），超过后自动切分新文件。
3. 日志保留天数必须可配置（建议默认 7 天），超过保留期的日志文件必须自动清理。
4. 日志轮转后的文件命名须包含日期或序号，便于排序检索。
5. 推荐使用 `pino-roll` 或 `winston-daily-rotate-file` 实现日志轮转，配置示例（winston）：
   ```javascript
   const transport = new DailyRotateFile({
     filename: 'logs/app-%DATE%.log',
     datePattern: 'YYYY-MM-DD',
     maxSize: '100m',
     maxFiles: '7d',
     zippedArchive: true,
   });
   ```

检查方式：日志格式审查 + CI lint
阻断级别：阻断合并

---

## 链路追踪（MUST）

1. 必须集成 OpenTelemetry SDK（`@opentelemetry/sdk-node`），统一追踪、指标和日志关联。
2. 每个入站请求必须创建根 Span，包含 `http.method`、`http.url`、`http.status_code` 属性。
3. 跨服务调用（HTTP/gRPC/消息队列）必须传播 `traceparent` 上下文头，确保分布式链路完整。
4. 关键业务操作（数据库查询、Redis 操作、外部 API 调用）必须创建子 Span。
5. Span 属性中禁止包含敏感数据（密码、令牌、个人身份信息）。
6. 链路数据必须导出到 Jaeger/Zipkin/OTLP Collector，禁止仅在本地打印。

### SHOULD
1. 推荐使用 `@opentelemetry/auto-instrumentations-node` 自动埋点常见库（Express、Fastify、Prisma、ioredis、pg 等）。
2. 推荐将 `traceId` 注入日志上下文，实现日志与链路一键关联。
3. 推荐为慢操作（>1s）设置告警采样率。

检查方式：链路数据完整性测试 + 监控平台验证
阻断级别：告警记录

---

## 指标监控（MUST）

1. 必须暴露 Prometheus 格式的 `/metrics` 端点（推荐使用 `prom-client`）。
2. 必须采集以下基础指标：
   - `http_requests_total`：HTTP 请求总量（标签：`method`、`path`、`status_code`）
   - `http_request_duration_seconds`：请求耗时直方图
   - `nodejs_eventloop_lag_seconds`：Event Loop 延迟
   - `nodejs_active_handles_total`：活跃句柄数
   - `nodejs_heap_size_bytes`：堆内存使用量
   - `process_cpu_seconds_total`：CPU 使用时间
3. 业务关键指标必须自定义采集（如订单创建量、支付成功率、消息队列积压数）。
4. 指标标签（label）值必须有限且可枚举，禁止使用 userId、orderId 等高基数值作为标签。
5. `/metrics` 端点必须与业务 API 端口分离或配置访问控制，禁止公网无鉴权暴露。

### SHOULD
1. 推荐为数据库连接池使用量、Redis 连接数、消息队列积压量配置独立指标。
2. 推荐使用 Grafana 仪表盘可视化关键指标，并配置告警规则。
3. 推荐定义 SLI/SLO：如 P99 延迟 < 500ms，错误率 < 1%。

检查方式：Prometheus 抓取验证 + Grafana 仪表盘
阻断级别：告警记录

---

## 审计日志（MUST）

1. 用户关键操作（登录/登出、权限变更、数据删除、配置修改）必须记录审计日志。
2. 审计日志必须包含：操作人、操作时间、操作类型、目标资源、变更前后值、来源 IP。
3. 审计日志必须持久化存储（数据库或独立日志通道），禁止仅输出到控制台日志。
4. 审计日志禁止篡改和删除，推荐使用 append-only 存储。

### SHOULD
1. 推荐使用 NestJS Interceptor 自动采集审计日志，减少业务代码侵入。
2. 推荐审计日志与业务日志分离存储，便于独立查询和合规审计。

检查方式：审计日志完整性审查
阻断级别：阻断合并
