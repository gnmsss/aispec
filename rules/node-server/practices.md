# Node.js 服务端场景实践(按需加载)

硬约束与红线见 `baseline.md`。默认技术栈 NestJS + Prisma;Express/Fastify 项目按同等分层与约束执行。

## 项目结构(NestJS 模块化单体)

```text
src/
├── main.ts                  # 仅启动与组装
├── app.module.ts            # 根模块
├── common/                  # 无业务语义:decorators、filters(全局异常)、guards、interceptors(响应包装/日志)、
│   │                        # pipes、middleware(request_id)、constants(error-codes、cache-keys)、interfaces、utils
├── config/                  # 配置命名空间(app/database/redis/jwt)+ env.validation.ts(Schema 校验)
├── infra/                   # database(prisma.service)、cache(redis.service)、storage、queue、logger、health
└── modules/<module>/        # <m>.module.ts、<m>.controller.ts、<m>.service.ts、<m>.repository.ts、dto/、entities/
```

1. admin 与 client 等作用域独立 controller + 独立 Guard 链,禁止 if 分支混合鉴权。
2. 模块间禁止直接调用对方 repository;跨模块经 service 接口(NestJS 模块 exports 显式声明)。

## API 契约

1. OpenAPI 用 `@nestjs/swagger` 装饰器生成(Code-First 单一契约源):DTO 必须完整标注,响应模型显式声明。
2. API 变更同 PR 确认 OpenAPI 无非预期漂移;破坏性变更评审升版本(`/api/v1`)。
3. 全局 `ValidationPipe`(`whitelist: true, transform: true`)+ class-validator/zod 完成入参校验。

## 组件初始化与生命周期

1. 基础组件封装为 infra 模块(全局 `@Global()` 或显式导入),经 DI 注入;PrismaClient/ioredis 单例由模块管理。
2. 生命周期钩子:`onModuleInit` 建连、`onApplicationShutdown` 断连;`app.enableShutdownHooks()` 启用优雅停机,响应 SIGTERM 排空在途请求。
3. 健康检查用 `@nestjs/terminus`:`/healthz`(存活)与 `/readyz`(就绪,检查 DB/Redis)。

## 数据库访问(Prisma)

1. ORM 统一选型(Prisma / Drizzle / TypeORM 之一),禁止混用;schema 变更走 `prisma migrate dev`(开发)/`prisma migrate deploy`(生产),迁移文件入库。
2. 查询必须 `select`/`include` 显式指定字段与关联,禁止全字段;批量用 `WHERE IN`/`createMany`,禁止循环单条(N+1)。
3. 事务:service 层 `prisma.$transaction(async (tx) => {...})` 交互式事务,设超时(默认 ≤ 30s);事务内禁止外部 API 调用与消息发送,用「先写库后异步」模式。
4. 破坏性迁移(删列、改类型)分阶段执行:先兼容双写,再切换,后清理;生产迁移前在预发验证。
5. 慢查询:开发环境开启 query log 排查;生产按阈值(默认 200ms)记 WARN 脱敏日志。
6. 金额用 `Decimal`;分页强制且限制 pageSize 上限。

## 缓存

1. 分布式用 Redis(ioredis 封装于 `infra/cache`);键集中定义 `constants/cache-keys.ts`,格式 `{服务}:{业务域}:{标识}`;全部设 TTL 加随机偏移。
2. Cache-Aside:先写库后删缓存,删除失败有补偿;穿透用空值缓存/布隆过滤器;击穿热点键用分布式锁或逻辑过期;强一致数据禁止以缓存为数据源。
3. Redis 分布式锁设超时 + 唯一标识防误释放。

## 后台任务与队列

1. 可靠任务队列统一 BullMQ(Redis)或平台等效;`@nestjs/schedule` 定时任务多实例必须配分布式锁。
2. 任务幂等(唯一 jobId / 去重键),执行结果持久化;失败重试 ≤ 3 次带退避 + 死信处理 + 积压告警。
3. 长任务支持取消与超时;Worker 独立进程部署,与 API 进程分离。

## 文件存储

1. 统一对象存储(MinIO/OSS/S3,`@aws-sdk/client-s3` 兼容接口),封装于 `infra/storage`;禁止存本地磁盘。
2. 上传校验大小 + MIME/文件头白名单;流式处理(禁止整读内存);存储名 UUID 重命名;元信息入库。
3. 访问分级:公开(CDN)/受限(签名 URL ≤ 30min)/敏感(SSE + 短签名 + 审计),不同级别不同 Bucket。

## 可观测性

1. OpenTelemetry SDK(auto-instrumentations-node)接入 HTTP/Prisma/ioredis,W3C Trace Context 透传。
2. 指标:RED + 下游依赖 + 事件循环延迟(event loop lag)、堆内存、GC;Prometheus 格式暴露于管理端口。
3. 告警必配:错误率 >1%、P99 >1s、实例不可用、事件循环阻塞、队列积压;分级 P0/P1/P2 并指定负责人。

## 性能

1. 响应时间目标 P95 ≤ 200ms、P99 ≤ 500ms;耗时操作(导出、批量)入队列异步化返回任务 ID。
2. 大 JSON/大文件流式处理(`stream.pipeline`);高吞吐场景考虑 Fastify 适配器;响应启用压缩。
3. 性能排查:`clinic.js`/`0x`/Chrome DevTools inspector(仅非生产);压测用 `autocannon`/`k6`。

## 测试

1. 单元测试:Vitest 或 Jest(项目统一);service mock repository,覆盖正常/边界/异常;修复缺陷补回归测试。
2. repository 集成测试用 Testcontainers 起真实数据库;API 层用 Nest `TestingModule` + supertest。
3. 测试禁止依赖外部真实服务;测试数据用 factory 构造。

## 微服务差异(microservice profile)

1. 每服务独立部署;禁止共享 ORM schema/实体包,服务间只通过契约(OpenAPI/proto)通信。
2. 同步调用统一 HTTP(fetch/undici)或 gRPC(`@grpc/grpc-js`);跨服务一致性用 Saga/Outbox,禁止分布式大事务。
3. 弹性:下游调用超时、重试、熔断(如 `cockatiel`/服务网格);超时逐层递减。
4. 消息:统一 broker(Kafka/RabbitMQ/Redis Streams);消费幂等 + 死信 + 积压告警;消息头透传 trace 上下文。
5. 容器:多阶段构建(pnpm fetch 缓存层 + `node:22-slim`)、非 root 运行;K8s 配探针与资源限制;发布走灰度。
