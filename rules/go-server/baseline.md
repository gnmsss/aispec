# Go 服务端基线(必载)

适用:所有 Go 服务端代码(HTTP API、gRPC、消息消费、定时任务、Worker)。
冲突优先级:`rules/database.md` > `rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Go 1.25+,以 `go.mod` 的 `go`/`toolchain` 指令为准;升级 Go 版本单独提交。
2. 依赖统一走 Go Modules,`go.sum` 必须入库;提交前 `go mod tidy` 后无额外变更。
3. CI 必须集成 `govulncheck`,高危漏洞(CVSS ≥ 7.0)阻断合并;商用项目禁止引入 GPL/AGPL 依赖;禁止引入已归档或 12 个月以上无维护的依赖。
4. 依赖更新与业务代码分开提交。
5. 合并门禁:`gofmt`、`go vet`、`golangci-lint`、`go test ./...` 全部通过;并发代码至少跑一次 `-race`。

## 分层架构(MUST)

1. 依赖单向:`transport -> service -> repository`,禁止反向依赖与循环依赖;`main/bootstrap` 只做组装与生命周期管理。
2. `transport` 只做协议适配(解析、校验、调用 service、响应映射),禁止直接访问数据库/缓存/对象存储/消息中间件,禁止绕过 service 调 repository。
3. `service` 承载用例编排、事务边界、幂等策略、领域规则,禁止依赖 HTTP/gRPC 框架类型和协议层 DTO,禁止直接写 SQL/ORM 细节。
4. `repository` 只做数据访问与持久化映射,禁止业务决策、鉴权、跨聚合编排。
5. 协议层 DTO 不下沉;持久化模型不直接透传到对外 API;层间转换显式实现。
6. 跨层函数第一个参数必须是 `context.Context`。
7. 通用无业务语义能力放 `pkg/`(如 `pkg/middleware`、`pkg/errkit`);带作用域语义的能力放 `internal/`,一文件一责任,禁止 `util.go`/`common.go`/`misc.go`。

## 错误处理(MUST)

1. 业务错误与系统错误分离,可用 `errors.Is`/`errors.As` 判断;业务错误必须携带结构化错误码(如 `USER_NOT_FOUND`),禁止纯数字码、禁止裸 `errors.New` 承载业务语义。
2. 业务错误集中声明为预定义变量(`internal/<module>/domain/*_error.go`);`fmt.Errorf("%w")` 仅用于添加调用上下文,禁止临时创建业务错误。
3. 禁止吞错;每层返回错误加调用上下文前缀并用 `%w` 保留根因。
4. HTTP/gRPC 入口必须有统一错误处理中间件 + panic recover(记录 `debug.Stack()` 并告警);禁止在多个 handler 分散手写错误映射;业务代码禁止用 `panic` 替代 `return err`。
5. 系统错误(驱动错误、RPC 原始错误、堆栈)必须记录日志但禁止原样返回给调用方。

## API 响应(MUST)

1. HTTP 响应统一包结构:`code`、`message`、`data`、`request_id`、`timestamp`;`request_id` 由中间件注入。
2. 语义化状态码:2xx 成功、4xx 客户端/业务错误、5xx 系统错误;禁止失败请求统一返回 200 再靠业务 code 区分。
3. 错误 `message` 必须是可控文案,禁止包含 SQL、堆栈、内部地址、密钥。
4. 对外时间统一 UTC(ISO 8601 或 Unix 时间戳);HTTP API 必须版本化(`/api/v1`),破坏性变更升版本。
5. 列表接口必须分页并限制 `page_size` 上限;写接口必须定义幂等策略。

## 配置(MUST)

1. 配置目录 `configs/`,`application.yml` + `application-<profile>.yml`;profile 显式指定并白名单校验(`dev/test/staging/prod`),启动日志输出当前 profile。
2. 密钥来自环境变量或密钥管理服务,禁止入库、禁止硬编码;外部依赖(DB/Redis/对象存储)的地址、凭据、超时、连接池参数全部走配置。
3. 数据库必须显式声明 `type`(仅 `mysql` 或 `postgresql`);需求未指定时先询问,禁止自行假设。
4. CORS 白名单从配置加载、支持多域名;`allow_credentials=true` 时禁止 `allowed_origins: "*"`。
5. 启动阶段完成配置校验,失败快速退出并输出明确错误。

## 日志(MUST)

1. 统一使用结构化日志(标准库 `log/slog` 或项目统一封装),禁止 `fmt.Print*`/标准库 `log.Print*` 打日志或调试(CI 用 `golangci-lint` + `forbidigo` 阻断)。
2. 每条请求链路记录 `request_id`、`trace_id`,有用户上下文时记录 `user_id`。
3. 禁止输出密码、令牌、证件号、银行卡等敏感信息;组件日志(连接串等)必须脱敏。
4. 生产默认 `INFO` 级;输出目标可配置;输出到文件时必须配置轮转(大小、保留期)。

## 红线(MUST NOT)

1. 禁止在 handler 中直接操作数据库。
2. 禁止忽略错误返回值(显式注释说明的可忽略场景除外)。
3. 禁止无上限缓存、无上限队列、无超时外部调用;所有 I/O(DB、HTTP、RPC、MQ)必须设超时。
4. 禁止未评审的破坏性接口变更和数据库结构变更;数据库结构变更遵循 `rules/database.md`。
5. 禁止在 `init()` 中建立数据库、Redis 等外部连接;禁止包级全局可变单例(全局 `DB`/`RedisClient`);组件一律构造函数注入,禁止在 handler/service 直接构造第三方客户端。
6. 禁止生产环境启动时自动 `AutoMigrate`,迁移必须独立执行。
7. 禁止 `SELECT *`、禁止字符串拼接 SQL(必须参数化)、禁止无 `WHERE` 的更新/删除。
8. 禁止在业务端口暴露 pprof;pprof 绑定独立管理端口并鉴权。
