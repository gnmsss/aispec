---
name: node-server
description: Node.js 服务端工程规范。编写或修改 Node.js 服务端代码(NestJS/Express/Fastify API、BullMQ 任务、Worker)、审查 Node.js 服务端代码变更、或初始化 Node.js 服务端项目时使用。
---

# Node.js 服务端规范

## 编码引导

1. 任何 Node.js 服务端编码任务,先读 `rules/node-server/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/node-server/practices.md` 对应章节:项目结构、API 契约、组件初始化、Prisma 数据访问、缓存、后台任务、文件存储、可观测性、性能、测试;微服务项目额外读文末「微服务差异」。
3. 跨域联动:
   - API 契约变更 / 前后端联调 → `rules/collaboration.md`
   - 数据库 schema / 迁移 → `rules/database.md`(优先级最高)
   - 认证、密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:controller → service → repository 单向;controller 未直接操作数据库;service 无 HTTP 框架类型
2. 类型:无 `any`/`as any`/`@ts-ignore`;strict 全开;DTO 有校验
3. 异步:无 floating promise;请求路径无同步 I/O;无事件循环阻塞;无 setTimeout 定时任务
4. 异常:无空 catch;统一 ExceptionFilter/错误中间件;系统错误不外泄
5. 响应:统一包结构,语义化状态码,无 200 兜底
6. ORM:查询有 select/include;无 N+1;事务在 service 层且无外部调用;金额用 Decimal
7. DI:组件注入获取,无顶层建连,无全局实例,无手动 new 客户端
8. 配置:Schema 校验,无散读 process.env,无硬编码密钥;CORS 走配置
9. 日志:结构化,无 console/debugger 残留,无敏感信息
10. 迁移:走迁移文件,未修改历史迁移,未启动时自动执行
11. 测试:有断言,不依赖外部真实服务;缺陷修复有回归测试
12. 契约:API 变更同 PR 更新 OpenAPI;破坏性变更已评审

## 项目脚手架

初始化新 Node.js 服务端项目时:

1. 确认(未指定必须先询问):框架(NestJS/Fastify)、数据库类型(mysql/postgresql)、ORM(Prisma/Drizzle)。
2. Node 22+ LTS + pnpm + TypeScript strict + ESM;配置 ESLint flat config + Prettier + husky + lint-staged,`engines`/`.nvmrc` 锁版本。
3. 按 `rules/node-server/practices.md`「项目结构」建 `src/{common,config,infra,modules}`。
4. 落地骨架:统一响应拦截器、全局异常过滤器、request_id 中间件、env Schema 校验、pino 结构化日志、terminus 健康检查、优雅停机钩子。
5. 首个业务模块按 `modules/<module>/{module,controller,service,repository,dto}` 打样,配 Vitest/Jest 测试。
