# 运维基线:可观测性与环境管理(跨域)

适用:所有服务端与需要监控/多环境的应用。各域 practices 中的可观测性条款为域内细则,本文件为跨域基线。

## 指标(Metrics)(MUST)

1. 黄金指标全覆盖:流量(Rate)、错误率(Errors)、时延(Duration,P95/P99)、饱和度(Saturation)。
2. 命名:小写下划线 + 单位后缀(`http_request_duration_seconds`)+ 来源前缀(`app_`/`biz_`/`infra_`);计数器 `_total`。
3. 指标格式 Prometheus 兼容,暴露于管理端口。

## 日志(Logging)(MUST)

1. 结构化(JSON/key-value),含时间戳、级别、服务名、`trace_id`/`request_id`;错误日志带上下文(用户、操作)。
2. 生产默认 INFO;禁止敏感信息(密码、Token、证件号、银行卡);保留:在线 ≥ 30 天,归档 ≥ 180 天。

## 追踪(Tracing)(MUST)

1. 统一 OpenTelemetry SDK,W3C Trace Context 跨服务传播(HTTP Header / gRPC metadata / MQ 消息头)。
2. 关键操作建 Span:HTTP、数据库、缓存、消息、外部 API;命名 `HTTP GET /api/orders/:id`、`DB SELECT orders`、`CACHE GET user:`、`MQ PUBLISH order.created` 风格。
3. 错误与慢请求强制采样;常规采样率可配置(生产 1%–10%)。

## 告警(MUST)

1. 分级:P0(服务不可用/数据丢失,即时电话级)、P1(核心指标劣化,即时消息)、P2(非核心,工作时间);每条告警有负责人与响应 SLA(P0 ≤ 15 分钟,P1 ≤ 1 小时)。
2. 必配:错误率 > 阈值、P99 超标、实例不可用、依赖(DB/缓存/MQ)异常、队列积压。
3. 抑制:5 分钟去抖、同根因聚合、发布窗口静默非 P0。

## SLO(SHOULD)

1. 核心服务定义 SLI/SLO(可用性、延迟)与错误预算策略;预算耗尽时冻结非必要发布。
2. 必备仪表盘:服务概览(黄金指标 + 版本 + 部署)、错误详情、依赖健康度、业务看板。

## 环境管理(MUST)

1. 标准环境:`dev / test / staging / prod`,配置隔离,禁止共享数据库与密钥;环境名白名单校验。
2. 配置注入:环境变量或配置中心;密钥走密钥管理(参见 `rules/security.md`);禁止硬编码环境差异参数;应用启动日志输出当前环境。
3. staging 尽可能镜像 prod(拓扑、版本、数据结构);生产数据入非生产环境必须脱敏(参见 `rules/database.md`)。
4. 环境提升顺序:dev → test → staging → prod,禁止跳级发布;prod 变更走发布窗口与审批。

## 功能开关(SHOULD)

1. 风险功能上线配开关,支持不发版关闭;开关有负责人与回收时间,禁止长期存留死开关。
2. 开关状态变更记录审计;灰度放量经开关或发布平台按比例控制。

## 发布与回滚(MUST)

1. 生产发布支持健康检查、优雅停机、失败自动/一键回滚;版本可追溯(镜像 tag / commit)。
2. 数据库迁移与应用发布解耦(先兼容迁移后发应用);回滚方案发布前明确。
