# rules/go-server/common/error-handling.md

## 错误分类（MUST）
1. 错误必须区分业务错误与系统错误，调用方可通过 `errors.Is` 或 `errors.As` 判断。
2. 外部依赖错误必须带上依赖标识和操作上下文，便于定位。
3. 用户可见错误信息必须可控，禁止泄露 SQL、密钥、内部路径。
4. 业务错误必须以预定义变量形式集中声明在 `*_error.go` 文件中（如 `var ErrUserNotFound = errkit.New(UserErrCodeNotFound, "用户不存在")`），禁止在 service/repository 中通过 `fmt.Errorf` 临时创建业务错误。
5. `fmt.Errorf` + `%w` 仅允许用于添加调用上下文包装已有预定义错误（如 `fmt.Errorf("repository.UpdateSession: %w", err)`），不允许用于定义新的业务错误语义。

## 错误类型建模（MUST）
1. 自定义错误类型必须实现 `error` 接口，推荐同时实现 `Unwrap() error` 以支持 `errors.Is` / `errors.As` 链式判断。
2. 业务错误类型至少包含：错误码（`Code`）、用户可见消息（`Message`）、可选的根因（`cause error`）。
3. 禁止使用裸字符串 `errors.New("xxx")` 定义带业务语义的错误，必须携带结构化错误码。
4. sentinel error（如 `var ErrNotFound = ...`）必须为包级不可变变量，禁止在运行时修改。

### 错误类型示例
```go
type BizError struct {
    Code    string
    Message string
    cause   error
}

func (e *BizError) Error() string { return e.Message }
func (e *BizError) Unwrap() error { return e.cause }

func NewBizError(code, message string) *BizError {
    return &BizError{Code: code, Message: message}
}

func (e *BizError) Wrap(err error) *BizError {
    return &BizError{Code: e.Code, Message: e.Message, cause: err}
}
```

## 错误归属与目录（MUST）
1. `pkg` 只放通用错误机制（如错误类型、错误码类型、`Wrap` 辅助），不放业务语义错误。
2. 带业务语义的错误必须放在 `internal` 作用域目录，并按作用域拆文件（例如 `user_error.go`、`order_error.go`）。
3. `system_error.go` 仅用于系统级分类或封装，不得直接作为对外响应内容返回。
4. 推荐避免使用包名 `errors`，应使用 `errkit`、`apperr`、`errcode` 等避免与标准库歧义。

## 错误传播（MUST）
1. 禁止吞错；返回错误时必须保留根因，统一使用 `%w` 包装。
2. 系统错误必须记录日志（最少包含 `request_id`、操作、依赖标识、根因），但禁止原样透传给调用方。
3. 边界层（HTTP/gRPC/消息）必须通过统一错误处理中间件将内部错误映射成稳定业务错误码与可控消息。
4. `panic` 只允许用于不可恢复场景，必须由统一恢复中间件处理并报警。
5. 禁止在多个 handler 中重复手写错误映射逻辑，错误到响应的转换必须集中治理。
6. 每一层返回错误时必须添加调用上下文前缀（`fmt.Errorf("repo.GetUser: %w", err)`），确保日志可追溯到具体调用链。

## 统一错误处理中间件（MUST）
1. HTTP 服务必须有全局错误处理中间件，拦截 handler 返回的错误并映射为标准响应结构。
2. 中间件必须区分 `BizError`（映射为业务错误码 + 用户可见消息）和非业务错误（映射为通用系统错误码 + 固定消息）。
3. 非业务错误必须记录完整日志（含 stack trace 或调用链），响应中禁止暴露内部细节。
4. gRPC 服务必须使用 `UnaryInterceptor` / `StreamInterceptor` 实现等效的错误拦截和 `status.Error` 映射。

### HTTP 错误中间件示例
```go
func ErrorHandlerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        if len(c.Errors) == 0 {
            return
        }
        err := c.Errors.Last().Err
        var bizErr *BizError
        if errors.As(err, &bizErr) {
            c.JSON(http.StatusOK, Response{Code: bizErr.Code, Message: bizErr.Message})
        } else {
            log.Error("unhandled_error", "request_id", c.GetString("request_id"), "error", err)
            c.JSON(http.StatusInternalServerError, Response{Code: "SYSTEM_ERROR", Message: "服务异常，请稍后重试"})
        }
    }
}
```

## Panic 恢复（MUST）
1. 必须在 HTTP/gRPC 入口注册 recover 中间件，捕获所有 panic 并转为错误响应。
2. recover 捕获后必须记录 panic 堆栈（`debug.Stack()`）和请求上下文。
3. recover 后必须触发告警（告警规则参见 `common/observability.md`），不允许静默恢复。
4. 业务代码中禁止使用 `panic` 替代 `return err`；`panic` 仅允许在程序初始化阶段不可恢复时使用。

## 错误码治理（MUST）
1. 同一服务内错误码唯一且语义稳定。
2. 新增错误码必须补充文档和测试。
3. 业务错误码应按作用域分段管理，避免 `user`、`order`、`system` 等作用域互相混用或冲突。
4. 错误码格式推荐：`{作用域}_{类别}_{序号}`（如 `USER_NOT_FOUND`、`ORDER_AMOUNT_INVALID`）。
5. 禁止使用纯数字错误码，必须具备可读语义。

## 错误响应结构（MUST）
1. HTTP API 错误响应必须遵循统一的 JSON 结构，至少包含 `code`（错误码）和 `message`（用户可见消息）。
2. 可选字段：`details`（字段级校验错误列表）、`request_id`（请求标识）。
3. 校验错误（如参数缺失、格式错误）必须返回 HTTP 400 + 字段级错误详情。
4. 鉴权错误返回 HTTP 401 / 403，禁止返回 200 + 业务错误码。

检查方式：代码审查 + 单元测试（验证错误映射）
阻断级别：阻断合并
