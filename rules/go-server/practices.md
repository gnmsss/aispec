# Go 服务端场景实践(按需加载)

硬约束与红线见 `baseline.md`。本文件覆盖:项目结构、API 契约、组件初始化、数据访问、缓存、并发、任务、文件存储、可观测性、性能、测试;微服务差异见文末。

## 项目结构(模块化单体)

```text
cmd/server/main.go              # 仅启动与依赖组装
configs/                        # application.yml + application-<profile>.yml
docs/api/openapi.yaml           # HTTP 契约
pkg/
├── middleware/                 # 跨作用域技术中间件:cors、request_id、logging、recovery、ratelimit
└── errkit/                     # 通用错误类型、错误码类型、Wrap 辅助(无业务语义)
internal/
├── app/bootstrap/              # container.go、providers.go、shutdown.go
├── app/http/middleware/<scope>/ # 作用域中间件:admin/auth.go、user/auth.go(独立中间件链,禁止 if 分支混合)
├── platform/<component>/       # logger、database、gorm、redis、minio、jwt 等组件初始化
└── modules/<module>/
    ├── domain/                 # 领域对象 + <module>_error.go 业务错误
    ├── service/                # 用例编排、事务边界
    ├── repository/model/       # 持久化模型(常规读写)
    ├── repository/query/       # 临时读模型(仅统计/报表,禁止用于写入)
    └── transport/http|dto/
```

1. 模块间禁止直接调用对方 repository;跨模块协作走 service 接口或模块 Facade。
2. `internal/shared` 只放无业务语义的技术组件,禁止放业务实体。
3. 持久化模型与读模型按职责独立文件,禁止 `models.go` 汇总文件。

## API 契约

1. HTTP 契约用 OpenAPI 3.x(单体放 `docs/api/openapi.yaml`,微服务放 `api/openapi/`);gRPC 用 proto3,显式版本命名空间(`order.v1`)。
2. 每个服务只选一种契约主来源(Contract-First 或 Code-First),禁止双源维护;API 变更与契约文件同一 PR 更新。
3. 契约变更标记兼容级别(兼容/条件兼容/破坏性);破坏性变更必须评审。
4. 对外字段语义稳定,禁止透传数据库字段名或内部枚举。
5. 新建资源返回 `201`,无响应体删除返回 `204`;校验错误返回 400 + 字段级 `details`;鉴权错误 401/403。

## 组件初始化与生命周期

1. 手动 DI(构造函数注入)为默认;可用 `google/wire` 等编译期注入;组装根在 `cmd/*/main.go` 或 `internal/app/bootstrap`。
2. 初始化顺序:`config -> logger -> metrics/tracing -> db -> cache -> 存储 -> repository -> service -> transport`;失败默认 fail fast;关闭顺序反向,退出有超时控制。
3. 每个组件包提供 `Config`、`New`,可关闭组件提供 `Close`,关键依赖提供 `Health`。
4. 提供 `/healthz`(存活,不依赖慢外部检查)与 `/readyz`(就绪,反映 DB 等关键依赖);非关键依赖故障可就绪但需降级标识与告警日志。
5. 优雅停机:收到 `SIGTERM` 先停接新请求,`http.Server.Shutdown` 排空在途请求;等待超时可配置,超时强退需输出告警与未完成数。

## 数据库访问

1. 访问方案唯一选型:GORM(模型驱动)或 sqlx(SQL 控制/性能敏感),禁止混用;选型记录在文档。
2. 连接池必须显式配置且走配置文件:`MaxOpenConns`、`MaxIdleConns`(≤ Open 的 50%)、`ConnMaxLifetime`(5–10min,短于 DB `wait_timeout`)、`ConnMaxIdleTime`;启动 `Ping`,停机 `Close`。
3. 事务边界在 service 层,用 `db.Transaction(func(tx) error)`;事务内禁止远程调用或不确定时延逻辑;嵌套用 SavePoint,禁止事务内再 `Begin()`。
4. 关联加载用 `Preload`/`Joins` 或批量查询,禁止循环内逐条查询(N+1);列表接口审查必查 N+1。
5. 批量写入单批 ≤ 500 条;GORM 保持 `AllowGlobalUpdate: false`。
6. 所有查询走 `context.WithTimeout`;`WHERE` 过滤字段必须有索引,新增查询附 EXPLAIN 或索引说明。
7. 慢查询阈值可配置(默认 200ms),超阈值 WARN 级记录(脱敏 SQL、耗时、影响行数、调用来源),接入统一日志通道;生产只记慢查询与错误查询。

## 缓存

1. 分布式共享数据用 Redis;本地缓存(`ristretto`/`bigcache` 等)仅用于读多写少、允许短暂不一致的场景;同类场景禁止混用多套方案。
2. 键规范 `{服务名}:{业务域}:{资源标识}`,集中定义(如 `internal/cache/keys.go`);禁止用户输入直接拼键。
3. 所有缓存设 TTL 并加 ±10% 随机偏移(防雪崩);数据变更主动删缓存(Cache-Aside:先写库后删缓存),删除失败必须有补偿(重试/异步队列)。
4. 穿透:空值缓存(短 TTL 30–60s)或布隆过滤器;击穿:`golang.org/x/sync/singleflight` 互斥回源或逻辑过期;雪崩:随机 TTL + Redis 高可用 + 降级限流兜底。
5. 金额、库存等强一致数据禁止以缓存为数据源;Redis 分布式锁必须设超时 + 唯一标识防误释放。
6. 命中率、回源次数、大键/热键纳入监控。

## 并发与资源

1. goroutine 必须可控:有退出条件、超时、错误回传;channel 由发送方关闭。
2. 并发写共享状态用明确同步机制;长任务支持 `context` 取消。
3. HTTP 客户端全局复用并配置 `Timeout`、`MaxIdleConnsPerHost`、`IdleConnTimeout`;`resp.Body` 必须 close 且读尽,保证连接复用。

## 定时任务与 Worker

1. 任务注册到统一任务管理模块,禁止业务代码散写 `time.Ticker`/`AfterFunc`。
2. 多实例定时任务必须分布式锁(Redis/DB 行锁/etcd 租约),锁带超时,标识含任务名+调度周期(`daily-report:2026-08-08`);抢锁失败静默跳过。
3. 任务必须幂等(状态检查或唯一键),执行结果持久化(任务名、时间、状态、耗时、影响行数)。
4. 失败记日志并告警;重试 ≤ 3 次带退避,超限转人工;长任务设 `context.WithTimeout`。

## 文件存储

1. 统一对象存储(MinIO/OSS/COS/S3),禁止存应用本地磁盘;SDK 封装在 `internal/platform/`,业务经接口调用。
2. 上传校验大小上限 + 类型白名单(按 MIME 和文件头,不只看扩展名);存储名用 UUID 重命名;>10MB 分片上传;元信息入库。
3. 访问分级:公开(Public Read Bucket + CDN)/受限(Private + 签名 URL ≤ 30min)/敏感(Private + SSE + 签名 URL ≤ 5min + 审计日志);不同级别不同 Bucket,禁止公开写。
4. 下载设置正确 `Content-Type`/`Content-Disposition`,大文件支持 Range;临时文件与未完成分片配置生命周期自动清理。

## 可观测性

1. 指标:RED(请求量、错误率、P95/P99 时延)+ 下游依赖成功率/耗时 + runtime(goroutine、GC、堆内存、连接池);Prometheus 格式 `/metrics` 绑定管理端口。
2. 追踪:OpenTelemetry SDK,W3C Trace Context(`traceparent`)透传(HTTP Header/gRPC metadata/MQ 消息头);Span 命名 `{服务}.{层}.{操作}`;错误与慢请求强制采样。
3. 告警必配:错误率 >1%、P99 >1s、实例不可用、goroutine 突增(>基线 3 倍)、连接池耗尽、消费延迟;分级 P0(即时电话)/P1(即时消息)/P2(工作时间),必须有负责人与响应 SLA。

## 性能

1. 热路径:预分配 slice/map 容量;循环内字符串拼接用 `strings.Builder`;高频大对象用 `sync.Pool`(Put 前重置)。
2. 容器部署设置 `GOMEMLIMIT` 配合内存限制;JSON 热路径可用 `sonic`/`go-json`;内部服务间通信优先 Protobuf;大文件流式处理。
3. 性能敏感模块写 Benchmark,热路径变更 PR 附前后对比;响应时间目标 P95 ≤ 200ms、P99 ≤ 500ms,耗时操作异步化(返回任务 ID)。

## 测试与发布

1. 新增/修改业务逻辑必须配套测试:service 单元测试(表驱动,mock 依赖)、repository 集成测试(验证 SQL/事务/索引)、transport 契约测试(状态码、错误映射、响应结构)。
2. 修复缺陷必须补回归测试;覆盖成功路径、参数错误、下游失败、超时/取消。
3. 发布支持健康检查、优雅停机、失败回滚;验证过探针行为与停机排空。

## 微服务差异(microservice profile)

1. 结构:每服务独立 `go.mod` 或 monorepo 明确边界;契约源 `api/openapi/` + `api/proto/`;禁止共享 ORM model,服务间只通过契约通信。
2. 同步调用统一 gRPC 或 HTTP 二选一为主;跨服务数据一致性用 Saga/Outbox,禁止分布式大事务。
3. 服务注册发现 + 健康检查驱动负载均衡;入口统一 API 网关(鉴权、限流、路由,不写业务逻辑)。
4. 弹性:下游调用必须限流、熔断、降级(如 `sony/gobreaker` 或服务网格能力);超时逐层递减。
5. 消息:消费必须幂等(唯一键/去重表),配置死信队列与积压告警;消息头透传 trace 上下文。
6. 服务间通信启用 mTLS 或服务网格等效机制;服务间认证独立于终端用户认证。
7. 配置中心(如 Nacos/Consul/etcd)管理动态配置与功能开关,热更新必须可回滚。
8. 容器:多阶段构建、非 root 运行、镜像含版本标签;K8s 配置 liveness/readiness 探针与资源 requests/limits;发布走灰度。
