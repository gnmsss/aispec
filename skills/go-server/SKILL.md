---
name: go-server
description: Go 服务端工程规范。编写或修改 Go 服务端代码(HTTP API、gRPC、消息消费、定时任务、Worker)、审查 Go 代码变更、或初始化 Go 服务端项目时使用。
---

# Go 服务端规范

## 编码引导

1. 任何 Go 服务端编码任务,先读 `rules/go-server/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/go-server/practices.md` 对应章节:项目结构、API 契约、组件初始化、数据库访问、缓存、并发、定时任务、文件存储、可观测性、性能、测试;微服务项目额外读文末「微服务差异」。
3. 跨域联动:
   - 涉及 API 契约变更 / 前后端联调 → 读 `rules/collaboration.md`
   - 涉及数据库 schema / 迁移脚本 → 读 `rules/database.md`(优先级最高)
   - 涉及认证、密钥、敏感数据 → 读 `rules/security.md`
4. 输出代码前自查红线清单(baseline.md「红线」节),违反任何一条不得输出。

## 代码审查清单

对 Go 服务端代码变更逐项检查,违反 MUST 项标记为阻断:

1. 分层:transport/service/repository 依赖单向,handler 未直接操作数据库
2. 错误:无吞错;`%w` 保留根因;业务错误用预定义变量;统一错误中间件,无散落映射
3. 响应:统一包结构,语义化状态码,无 200 兜底,错误消息不泄露内部细节
4. 超时:所有外部 I/O(DB/HTTP/RPC/MQ)有超时;查询走 `context.WithTimeout`
5. SQL:无 `SELECT *`、无拼接 SQL、更新删除有 WHERE、无 N+1、批量有上限
6. 事务:边界在 service,事务内无远程调用
7. 并发:goroutine 有退出条件;共享状态有同步;`resp.Body` close 且读尽
8. 配置与密钥:无硬编码地址/凭据;CORS 域名走配置
9. 日志:结构化,无 `fmt.Print` 调试残留,无敏感信息
10. 组件:无 `init()` 建连,无全局可变单例,依赖走构造函数注入
11. 测试:业务逻辑变更有配套测试;缺陷修复有回归测试
12. 契约:API 变更同 PR 更新 OpenAPI/proto;破坏性变更已评审升版本

## 项目脚手架

初始化新 Go 服务端项目时:

1. 确认三件事(未指定必须先询问):部署形态(单体/微服务)、数据库类型(mysql/postgresql)、HTTP 框架选型。
2. 按 `rules/go-server/practices.md`「项目结构」建目录:`cmd/` + `configs/` + `pkg/{middleware,errkit}` + `internal/{app/bootstrap,platform,modules}`。
3. Go 1.25+,初始化 `go.mod`;配置 `.golangci.yml`(启用 forbidigo 禁调试打印)、`configs/application.yml` + profile 文件。
4. 落地基础设施骨架:统一响应结构、错误中间件 + recover、request_id 注入、`/healthz` `/readyz`、优雅停机、slog 结构化日志。
5. 首个业务模块按 `modules/<module>/{domain,service,repository,transport}` 打样。
