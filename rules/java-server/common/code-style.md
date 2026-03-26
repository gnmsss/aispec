# rules/java-server/common/code-style.md

## 命名规范

### MUST
1. 类名使用 UpperCamelCase（大驼峰），如 `OrderService`、`UserController`。
2. 方法名和变量名使用 lowerCamelCase（小驼峰），如 `createOrder`、`userId`。
3. 常量使用全大写下划线分隔，如 `MAX_RETRY_COUNT`、`DEFAULT_PAGE_SIZE`。
4. 包名使用全小写，禁止下划线和大写字母，如 `com.example.order.service`。
5. 接口命名禁止以 `I` 开头（如禁止 `IUserService`），直接使用 `UserService`；实现类使用 `UserServiceImpl` 或语义化命名。
6. 布尔类型变量和方法使用 `is`/`has`/`can` 前缀，如 `isActive`、`hasPermission`。
7. DTO/VO/Request/Response 类必须带对应后缀，如 `OrderCreateRequest`、`OrderDetailVO`。
8. 文件命名必须与类名一致，禁止一个文件包含多个顶级 public 类。

### SHOULD
1. 方法命名体现动作语义：创建用 `create`、查询用 `find`/`get`、更新用 `update`、删除用 `delete`/`remove`。
2. 集合类型变量使用复数或 `List`/`Map`/`Set` 后缀，如 `orders`、`userIdList`。

## 注释规范

### MUST
1. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
2. 所有 public 类必须有 Javadoc 注释，说明类的职责和适用场景。
3. 所有 public 方法必须有 Javadoc 注释，包含 `@param`、`@return`、`@throws` 说明。
4. Entity/Model/DTO 类的所有字段必须有注释（Javadoc 或 `@Schema(description = "...")` 注解），说明字段含义与约束。持久化实体字段注释需包含数据库列含义。
5. 非 public 但逻辑复杂的方法（超过 30 行或包含复杂分支）必须添加中文注释。
6. 复杂业务逻辑、非直觉的条件判断、临时方案（workaround）必须行内注释说明背景和原因。
7. 禁止无意义注释（如 `// 获取用户` 后面跟 `getUser()`），注释必须提供代码本身未表达的信息。
8. 接口方法注释必须说明实现方的职责约束和预期行为契约。

### SHOULD
1. TODO/FIXME 注释必须附带责任人和预计回收时间（如 `// TODO(zhangsan): 2026-04 迁移到新接口`）。
2. 注释随代码同步更新；代码逻辑变更后，对应注释必须同步修改，禁止过期注释残留。
3. 包级别注释（`package-info.java`）说明包的整体职责和主要用法。

检查方式：Checkstyle Javadoc 检查 + 人工审查
阻断级别：阻断合并

## 调试代码清理

### MUST
1. 禁止将 `System.out.println`、`System.err.println`、`e.printStackTrace()` 提交到主分支；所有日志输出必须通过 SLF4J 统一的结构化日志组件（参见 `common/observability.md`）。
2. 禁止将 `Thread.sleep()` 作为调试手段提交到主分支。
3. CI 阶段通过 SpotBugs 或自定义 Checkstyle 规则检测并阻断调试代码残留。
4. 开发环境允许临时使用调试打印，但提交前必须清理。

检查方式：SpotBugs + Checkstyle + CI 阻断
阻断级别：阻断合并

## Lombok 使用规范

### MUST
1. 允许使用 `@Getter`、`@Setter`、`@ToString`、`@EqualsAndHashCode`、`@Builder`、`@Slf4j`。
2. 禁止在 JPA Entity 上使用 `@Data`（会导致 `equals`/`hashCode` 问题），必须手动实现或使用 `@Getter` + `@Setter` 并手写 `equals`/`hashCode`。
3. 禁止使用 `@AllArgsConstructor` 配合 Spring Bean（破坏构造器注入的可读性），Spring Bean 必须显式编写构造函数。
4. `@Builder` 使用时必须配合 `@NoArgsConstructor(access = AccessLevel.PROTECTED)` + `@AllArgsConstructor(access = AccessLevel.PRIVATE)`，保证框架兼容性。
5. 禁止使用 `@SneakyThrows`，异常必须显式处理或声明。
6. 禁止使用 `val`/`var`（Lombok 的局部变量类型推断），避免降低代码可读性。

### SHOULD
1. 团队统一 Lombok 版本，纳入 BOM 管理。
2. 配置 `lombok.config` 限制可用注解范围。

## 分层编码要求
1. 分层依赖必须单向：`Controller → Service → Repository`，禁止反向依赖和循环依赖。
2. `Controller` 只负责协议适配：请求解析、参数校验（`@Valid`）、调用 `Service`、响应映射。
3. `Service` 负责用例编排、事务边界（`@Transactional`）、幂等策略、领域规则与权限策略。
4. `Repository` / `DAO` 只负责数据访问与持久化映射，不承载业务决策、鉴权策略或流程编排。
5. 启动类只做组装与生命周期管理，禁止承载业务逻辑。
6. 禁止在 Controller、启动类、Repository 之间跨层写业务捷径代码。

## 分层边界细则
1. `Controller` 禁止直接注入 `Repository`/`DAO`，必须经由 `Service` 调用。
2. `Controller` 禁止直接操作数据库、缓存、对象存储、消息中间件客户端。
3. `Service` 禁止依赖 `HttpServletRequest`/`HttpServletResponse` 等 Servlet API 类型。
4. `Service` 不得直接编写 SQL 或操作 EntityManager/SqlSession，数据访问必须通过 `Repository`。
5. `Repository` 禁止处理业务状态机、业务分支决策、跨聚合用例编排。
6. `domain` 模型与规则禁止依赖 `Controller`、`Repository`、`infrastructure` 的具体实现。

## 模型与 DTO 约束
1. 请求/响应 DTO 仅用于 `Controller` 层，禁止下沉到 `Service`、`Repository`。
2. JPA Entity / MyBatis Model 仅用于 `Repository` 层持久化映射，禁止直接透传到 API 响应。
3. 不同层模型转换必须显式实现（手动映射或 MapStruct），禁止在单个类上混用多层注解（如同时标注 `@Entity` 和 `@JsonProperty`）。
4. 领域对象用于表达业务语义，禁止直接透传持久化模型到外部 API。

## 事务、错误与日志边界
1. 事务边界定义在 `Service` 层（`@Transactional`），`Repository` 仅执行事务上下文内数据操作。
2. 错误处理遵循"内部系统异常记录、对外业务异常映射"原则。
3. `Controller` 层通过 `@ControllerAdvice` 将异常统一映射为稳定响应结构，禁止散落式手写映射逻辑。
4. `Repository` 层异常必须携带必要上下文（操作类型、关键参数），由 Service 或统一处理器转换。
5. 日志记录应在边界层和关键失败点进行，避免同一异常在多层重复打印。

## 代码可维护性
1. 每个方法只做一件事，单个方法建议不超过 50 行，避免超深嵌套分支（建议不超过 3 层）。
2. 公共代码先在模块内部复用，稳定后再考虑提升到共享模块。
3. 并发和资源生命周期必须在代码中可读可验证（超时、取消、关闭顺序）。
4. 对外行为变化必须有测试覆盖：至少覆盖成功路径、参数错误、下游失败、超时场景。

## 分层测试建议
1. `Service` 层优先单元测试，使用 Mockito mock 依赖验证用例编排和领域规则。
2. `Repository` 层优先集成测试（Testcontainers），验证 SQL/ORM 行为、事务一致性与索引假设。
3. `Controller` 层使用 `@WebMvcTest` 或 MockMvc 测试，验证状态码、错误映射和响应结构。
