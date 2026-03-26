# rules/node-server/common/code-style.md

## 命名规范

### MUST
1. 文件名使用 `kebab-case`（如 `user-service.ts`、`order.controller.ts`），禁止 `camelCase` 或 `PascalCase` 文件名。
2. 类名使用 `PascalCase`，接口名使用 `PascalCase`（禁止 `I` 前缀，如禁止 `IUserService`）。
3. 变量和函数使用 `camelCase`，常量使用 `UPPER_SNAKE_CASE`。
4. 枚举使用 `PascalCase`，枚举成员使用 `PascalCase` 或 `UPPER_SNAKE_CASE`。
5. 泛型类型参数使用有意义的名称（如 `TEntity`、`TResponse`），禁止单字母 `T`（除标准 `T extends` 场景外）。
6. 布尔类型变量和属性以 `is`、`has`、`should`、`can` 开头，如 `isActive`、`hasPermission`。
7. 异步函数必须返回 `Promise`，禁止返回 `void` 但内部执行异步操作（fire-and-forget）。

### SHOULD
1. 推荐文件按职责后缀命名：`*.controller.ts`、`*.service.ts`、`*.repository.ts`、`*.module.ts`、`*.dto.ts`、`*.entity.ts`。
2. 推荐 DTO 类名包含操作语义，如 `CreateUserDto`、`UpdateOrderDto`、`UserResponseDto`。

检查方式：ESLint `@typescript-eslint/naming-convention` + 代码审查
阻断级别：阻断合并

---

## TypeScript 类型注解

### MUST
1. 所有函数必须显式标注返回类型，禁止依赖类型推导（除简单箭头函数和回调外）。
2. 禁止使用 `any` 类型，必须使用 `unknown` + 类型收窄或泛型替代（通过 `@typescript-eslint/no-explicit-any` 强制）。
3. 禁止使用 `@ts-ignore`，必须使用 `@ts-expect-error` 并附注释说明原因。
4. DTO、Entity、配置对象必须定义完整的类型或接口，禁止使用内联对象类型传递。
5. API 响应类型必须独立定义，禁止在 controller 中使用 `Record<string, any>` 返回。
6. 联合类型超过 3 个成员时必须抽取为命名类型或枚举。

### SHOULD
1. 推荐使用 `satisfies` 操作符进行类型安全的对象字面量校验。
2. 推荐使用 `Readonly<T>`、`Required<T>`、`Partial<T>` 等工具类型表达意图。

检查方式：`tsc --noEmit` + ESLint
阻断级别：阻断合并

---

## Import 排序与模块组织

### MUST
1. import 必须按以下顺序分组，组间空行分隔：
   - Node.js 内建模块（`node:fs`、`node:path`）
   - 第三方依赖（`@nestjs/*`、`express`、`prisma`）
   - 项目别名导入（`@/modules/*`、`@/common/*`）
   - 相对路径导入（`./`、`../`）
2. 禁止使用 `require()` 导入模块（除动态加载或 CommonJS 互操作场景外），统一使用 ES Module `import`。
3. 禁止循环依赖，CI 必须配置 `eslint-plugin-import/no-cycle` 或 `madge` 检测。
4. 每个文件只导出一个主要职责，禁止 `util.ts`、`common.ts`、`misc.ts` 等模糊命名承载多责任逻辑。
5. barrel 文件（`index.ts`）仅允许在模块根目录使用，禁止嵌套 barrel 导出。

### SHOULD
1. 推荐使用 `eslint-plugin-import/order` 或 `@trivago/prettier-plugin-sort-imports` 自动排序。
2. 推荐 type-only import 使用 `import type` 语法（`@typescript-eslint/consistent-type-imports`）。

检查方式：ESLint + Prettier
阻断级别：阻断合并

---

## 注释规范

### MUST
1. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
2. 所有导出的类、函数、接口、类型必须有 JSDoc 注释，说明"做什么"和"为什么"。
3. Entity/Model/DTO 类的所有属性必须有注释（JSDoc 或装饰器 `@ApiProperty({ description: "..." })`），说明字段含义与约束。持久化实体属性注释需包含数据库列含义。
4. 复杂业务逻辑、非直觉的条件判断、临时方案（workaround）必须行内注释说明背景和原因。
5. 禁止无意义注释（如 `// 创建用户` 后面跟 `createUser()`），注释必须提供代码本身未表达的信息。
6. 接口和抽象类的注释必须说明实现方的职责约束和预期行为契约。

### SHOULD
1. TODO/FIXME 注释必须附带责任人和预计回收时间（如 `// TODO(zhangsan): 2026-04 迁移到新接口`）。
2. 注释随代码同步更新；代码逻辑变更后，对应注释必须同步修改，禁止过期注释残留。

检查方式：ESLint `jsdoc` 插件 + 人工审查
阻断级别：阻断合并

---

## 调试代码清理

### MUST
1. 禁止将 `console.log`、`console.debug`、`console.info`、`console.warn` 提交到主分支；所有日志输出必须通过项目统一的结构化日志组件（参见 `common/observability.md`）。
2. 禁止将 `debugger` 语句提交到主分支。
3. CI 阶段通过 ESLint 检测并阻断调试代码残留：
   - 启用 `no-console` 规则（可配置允许 `console.error` 用于启动阶段错误）。
   - 启用 `no-debugger` 规则。
4. 开发环境允许临时使用调试打印，但提交前必须清理。

检查方式：ESLint `no-console` + `no-debugger` + CI 阻断
阻断级别：阻断合并

---

## 分层编码要求

### MUST
1. 分层依赖必须单向：`controller → service → repository`，禁止反向依赖和循环依赖。
2. `controller` 只负责协议适配：请求解析、参数校验、调用 `service`、响应映射。
3. `service` 负责用例编排、事务边界、幂等策略、领域规则与权限策略。
4. `repository` 只负责数据访问与持久化映射，不承载业务决策、鉴权策略或流程编排。
5. 启动层（`main.ts` / `bootstrap`）只做组装与生命周期管理，禁止承载业务逻辑。
6. 禁止在 controller、main、repository 之间跨层写业务捷径代码。

---

## 分层边界细则

### MUST
1. `controller` 禁止直接访问数据库、缓存、对象存储、消息中间件客户端。
2. `controller` 禁止直接引用 `repository`，必须经由 `service` 调用。
3. `service` 禁止依赖 HTTP 框架类型（如 `Request`、`Response`）和协议层 DTO。
4. `service` 不得直接编写 ORM 查询代码，数据访问必须通过 `repository`。
5. `repository` 禁止处理业务状态机、业务分支决策、跨聚合用例编排。
6. `domain` 模型与规则禁止依赖 `controller`、`repository`、`infrastructure` 的具体实现。

---

## 模型与 DTO 约束

### MUST
1. 协议层 DTO 仅用于 `controller`，禁止下沉到 `service`、`repository`。
2. Entity 用于 ORM 映射和持久化操作，禁止直接透传到 API 响应。
3. 不同层模型转换必须显式实现（mapper 或 plainToInstance），禁止在单个类上混用多层装饰器。
4. DTO 必须使用 `class-validator` 或 `zod` 进行校验，禁止在 controller 中手写校验逻辑。
