# rules/python-server/common/code-style.md

## 命名与组织
1. 模块名、包名使用小写加下划线（snake_case），禁止驼峰命名。
2. 类名使用 PascalCase，函数和变量使用 snake_case，常量使用 UPPER_SNAKE_CASE。
3. 私有属性和方法以单下划线 `_` 前缀标识，禁止无理由使用双下划线 `__` 名称修饰。
4. 文件命名必须体现职责，禁止 `utils.py`、`common.py`、`misc.py` 等模糊命名承载多责任逻辑。
5. 每个模块 `__init__.py` 仅做导出声明，禁止在其中编写业务逻辑。

## 类型注解（MUST）
1. 所有公开函数和方法的参数与返回值必须添加类型注解。
2. 类属性必须使用类型注解声明。
3. 优先使用 Python 内置泛型语法（`list[str]`、`dict[str, int]`、`str | None`），Python 3.10+ 项目禁止使用 `typing.List`、`typing.Dict`、`typing.Optional`。
4. 复杂类型推荐定义 `TypeAlias` 或 `TypeVar`，提高可读性。
5. 函数返回值禁止使用裸 `dict` 或 `tuple` 作为公开 API 返回类型，必须使用 Pydantic model 或 `TypedDict`。

## import 排序（MUST）
1. import 顺序：标准库 → 第三方库 → 本项目模块，各组之间空行分隔。
2. 禁止通配符导入（`from xxx import *`），除 `__init__.py` 的显式重导出场景。
3. 禁止循环导入；必要时使用 `TYPE_CHECKING` 延迟导入。
4. import 排序由 `ruff`（isort 规则）自动管理，CI 中强制检查。

## Docstring 规范（MUST）
1. 所有公开类、函数、方法必须编写 docstring，说明职责、参数、返回值和异常。
2. docstring 风格统一使用 Google 风格或 NumPy 风格，项目内禁止混用。
3. ORM Model 和 Pydantic Schema 的所有字段必须有注释（docstring 或 `Field(description="...")`），说明字段含义与约束。持久化模型字段注释需包含数据库列含义。
4. docstring 语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
5. 禁止无意义 docstring（如 `"""创建用户"""` 后面跟 `def create_user()`），必须提供代码本身未表达的信息。
6. 接口/抽象类的 docstring 必须说明实现方的职责约束和预期行为契约。

### SHOULD
1. TODO/FIXME 注释必须附带责任人和预计回收时间（如 `# TODO(zhangsan): 2026-04 迁移到新接口`）。
2. 注释随代码同步更新；代码逻辑变更后，对应注释必须同步修改，禁止过期注释残留。

检查方式：`ruff` docstring 检查 + 人工审查
阻断级别：阻断合并

## 调试代码清理（MUST）
1. 禁止将 `print()`、`pprint()`、`breakpoint()`、`pdb.set_trace()` 等调试代码提交到主分支。
2. 所有日志输出必须通过项目统一的结构化日志组件（参见 `common/observability.md`）。
3. CI 阶段通过 `ruff` 规则检测并阻断调试代码残留：
   ```toml
   # pyproject.toml
   [tool.ruff.lint]
   select = ["T20"]  # flake8-print: 检测 print/pprint
   ```
4. 开发环境允许临时使用调试代码，但提交前必须清理。

检查方式：`ruff`（T20 规则）+ CI 阻断
阻断级别：阻断合并

## 分层编码要求
1. 分层依赖必须单向：`transport(router/view) -> service -> repository`，禁止反向依赖和循环依赖。
2. `transport` 只负责协议适配：请求解析、参数校验（Pydantic schema）、调用 `service`、响应映射。
3. `service` 负责用例编排、事务边界、幂等策略、领域规则与权限策略。
4. `repository` 只负责数据访问与持久化映射，不承载业务决策、鉴权策略或流程编排。
5. 启动层仅做组装与生命周期管理，禁止承载业务逻辑。
6. 禁止在 router、启动层、repository 之间跨层写业务捷径代码。

## 分层边界细则
1. `transport`（router/view）禁止直接访问数据库、缓存、对象存储、消息中间件客户端。
2. `transport` 禁止直接引用 `repository`，必须经由 `service` 调用。
3. `service` 禁止依赖 HTTP 框架类型（如 `Request`、`Response`）和协议层 DTO。
4. `service` 不得直接编写 SQL 或 ORM 查询细节代码，数据访问必须通过 `repository`。
5. `repository` 禁止处理业务状态机、业务分支决策、跨聚合用例编排。
6. `domain` 模型与规则禁止依赖 `transport`、`repository`、`platform` 的具体实现。

## 模型与 DTO 约束
1. Pydantic schema（请求/响应模型）仅用于 `transport`，禁止下沉到 `service`、`repository`。
2. ORM model（持久化模型）用于数据库交互，禁止直接透传到 API 响应。
3. 不同层模型转换必须显式实现，禁止在单个类上混用多层语义。

## 代码可维护性
1. 每个函数只做一件事，函数体建议不超过 50 行。
2. 避免超深嵌套分支（建议不超过 3 层），优先使用早返回（guard clause）模式。
3. 公共代码先在模块内部复用，稳定后再考虑提升到共享目录。
4. 对外行为变化必须有测试覆盖：至少覆盖成功路径、参数错误、下游失败场景。
