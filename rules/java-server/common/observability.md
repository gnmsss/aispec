# rules/java-server/common/observability.md

## 文档目标
1. 定义 Java 服务端日志、指标、链路追踪、告警的统一约束。
2. 微服务场景下的服务拓扑与全链路追踪增强内容一并覆盖。

---

## 日志（MUST）

1. 必须使用 SLF4J 作为日志门面，Logback 作为默认实现（Spring Boot 默认集成）。
2. 必须使用结构化日志（JSON 格式），通过 Logback 的 `LogstashEncoder` 或等效 JSON 编码器实现。
3. 每条请求链路必须记录 `requestId`、`traceId`，有用户上下文时记录 `userId`。通过 MDC（`MDC.put()`）注入。
4. 日志级别至少区分 `DEBUG`、`INFO`、`WARN`、`ERROR`。
5. 禁止输出敏感信息：密码、令牌、证件号、银行卡完整信息。
6. 日志必须包含时间戳（ISO 8601）、服务名（`spring.application.name`）、实例标识。
7. 生产环境默认日志级别为 `INFO`，`DEBUG` 级别仅在排查问题时临时开启。
8. 日志输出目标必须可配置（stdout / 文件 / 日志收集 Agent），通过 `logback-spring.xml` 按 Profile 区分。
9. 禁止使用 `System.out.println` 或 `e.printStackTrace()` 输出日志，必须通过 `@Slf4j` 或 `LoggerFactory.getLogger()` 获取 Logger。
10. Logger 必须使用参数化占位符（`log.info("user {} created", userId)`），禁止字符串拼接。

### SHOULD
1. 日志采集推荐统一收集到日志平台（ELK / Loki / 云日志服务），支持集中检索。
2. 关键操作日志（创建、删除、权限变更）单独标记，便于审计查询。
3. 日志量过大时支持采样策略（如 DEBUG 日志仅采样 10%）。

### 日志文件轮转（MUST，输出到文件时适用）
1. 当日志输出目标为文件时，必须配置日志轮转策略，禁止单文件无限增长。
2. 单个日志文件最大大小必须可配置（建议默认 100MB），超过后自动切分新文件。
3. 日志保留天数必须可配置（建议默认 7 天），超过保留期的日志文件必须自动清理。
4. 日志轮转后的文件命名须包含日期或序号，便于排序检索。
5. 推荐使用 Logback `RollingFileAppender` + `SizeAndTimeBasedRollingPolicy` 实现，配置示例：
   ```xml
   <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
       <file>logs/app.log</file>
       <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
           <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
           <maxFileSize>100MB</maxFileSize>
           <maxHistory>7</maxHistory>
           <totalSizeCap>2GB</totalSizeCap>
       </rollingPolicy>
       <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
   </appender>
   ```

检查方式：代码审查 + 日志平台配置审查
阻断级别：阻断合并

---

## 指标（MUST）

1. 必须使用 Micrometer 作为指标门面（Spring Boot Actuator 默认集成）。
2. 必须提供 RED 指标（Rate 请求量、Error 错误率、Duration 时延分位数 P95/P99）。
3. 下游依赖（数据库、Redis、第三方 API）必须有独立成功率和耗时指标。
4. 后台任务必须有吞吐量、失败数、重试次数指标。
5. 指标暴露格式推荐 Prometheus 兼容（`/actuator/prometheus` 端点），绑定管理端口。
6. 自定义业务指标（如订单量、支付成功率）必须通过 `MeterRegistry` 注册，禁止散写。

### 资源指标（MUST）
1. 以下运行时指标必须暴露（Micrometer 自动采集大部分）：
   - JVM：堆内存使用、GC 次数和停顿时间、线程数、类加载数。
   - 连接池：HikariCP 活跃连接数、空闲连接数、等待数。
   - 消息队列：消费延迟、积压量（微服务场景）。

检查方式：监控平台配置审查
阻断级别：阻断合并

---

## 链路追踪（MUST）

1. 必须接入分布式链路追踪系统（推荐 Micrometer Tracing + OpenTelemetry 或 Spring Cloud Sleuth + Zipkin）。
2. 追踪协议使用 **W3C Trace Context**（`traceparent` / `tracestate`），禁止自定义私有协议。
3. 跨服务调用必须透传 trace 上下文：
   - HTTP：通过 `traceparent` Header 传播（RestTemplate/WebClient 自动传播需配置 Interceptor）。
   - gRPC：通过 metadata 传播。
   - 消息队列：通过消息 Header/属性传播。
4. 关键业务流程必须具备端到端可追踪能力（从前端请求到最终数据库写入）。
5. Span 命名规范：`{服务名}.{层级}.{操作}`（如 `order-svc.service.createOrder`）。
6. Span 必须记录关键属性：HTTP 方法/路径/状态码、数据库语句（脱敏）、错误信息。

### SHOULD
1. 追踪采样率可配置（生产建议 1%-10%），高流量服务适当降低采样率。
2. 错误请求和慢请求强制采样（不受采样率限制），确保异常链路可追踪。
3. 追踪数据接入可视化平台（Jaeger / Tempo / Zipkin），支持链路检索和拓扑展示。

检查方式：集成测试（验证 trace 透传）+ 追踪平台审查
阻断级别：阻断合并

---

## 服务拓扑与依赖可视化（SHOULD）

1. 通过链路追踪数据自动生成服务调用拓扑图，展示服务间依赖关系。
2. 拓扑图必须可展示：调用方向、调用量、错误率、平均延迟。
3. 定期审查拓扑图，发现异常依赖（如循环调用、非预期调用链路）。
4. 新服务上线前必须在拓扑图中确认依赖关系符合架构设计。

检查方式：追踪平台拓扑审查
阻断级别：告警记录

---

## 告警规范（MUST）

1. 以下场景必须配置告警：
   - 接口错误率 > 阈值（建议 > 1%）。
   - 接口 P99 延迟 > 阈值（建议 > 1s）。
   - 服务实例不可用（健康检查失败）。
   - JVM 堆内存使用率 > 80%。
   - 线程数突增（> 基线 3 倍）。
   - 数据库连接池耗尽（HikariCP 等待超时）。
   - 消息队列消费延迟 > 阈值。
2. 告警必须分级：
   - **P0 严重**：服务不可用、数据丢失风险 → 即时通知（电话/即时消息）。
   - **P1 重要**：核心指标劣化、部分功能异常 → 即时消息通知。
   - **P2 一般**：非核心指标偏离、性能轻微劣化 → 工作时间通知。
3. 告警必须指定负责人，禁止无人认领的告警规则。
4. 告警触发后必须有响应 SLA（P0 ≤ 15 分钟、P1 ≤ 1 小时）。

### SHOULD
1. 告警抑制：同一故障触发的关联告警应聚合，避免告警风暴。
2. 告警自动创建事件工单，跟踪处理进度直至关闭。

检查方式：监控平台告警规则审查
阻断级别：阻断合并
