# rules/java-server/common/error-handling.md

## 异常分类

### MUST
1. 异常必须区分业务异常（`BusinessException`）与系统异常（`SystemException`），分别继承自统一基类。
2. 业务异常携带业务错误码和用户可见消息，可由调用方纠正（如参数错误、资源不存在）。
3. 系统异常表示内部故障（数据库不可用、第三方超时），不暴露给调用方。
4. 外部依赖异常必须带上依赖标识和操作上下文，便于定位。
5. 用户可见错误信息必须可控，禁止泄露 SQL、堆栈、内部路径、密钥。

## 异常类设计（MUST）

1. 统一定义基础异常类层次：
   - `BaseException`：所有自定义异常基类，包含 `errorCode`、`message`。
   - `BusinessException extends BaseException`：业务可纠正异常。
   - `SystemException extends BaseException`：系统内部异常。
2. 禁止直接抛出 `RuntimeException`、`Exception` 等通用异常，必须使用自定义异常类型。
3. 业务异常必须通过错误码枚举构造（如 `BusinessException.of(ErrorCode.USER_NOT_FOUND)`），禁止直接使用内联字符串构造（如 `new BusinessException("SOME_CODE", "随意消息")`）。
4. 每个异常实例必须包含 `errorCode`（String）和 `message`（String），可选包含 `data`（附加信息）。

## 统一异常处理（MUST）

1. 必须使用 `@ControllerAdvice` + `@ExceptionHandler` 实现全局统一异常处理。
2. 全局异常处理器必须至少覆盖以下异常类型：
   - `BusinessException` → 映射为对应 HTTP 状态码 + 业务错误码。
   - `MethodArgumentNotValidException` → `400` + 字段级校验错误信息。
   - `HttpMessageNotReadableException` → `400` + 请求体解析错误。
   - `NoHandlerFoundException` / `HttpRequestMethodNotSupportedException` → `404` / `405`。
   - `SystemException` → `500` + 通用错误消息（隐藏内部细节）。
   - `Exception`（兜底） → `500` + 通用错误消息 + 记录完整堆栈日志。
3. 禁止在 Controller 中手写 try-catch 做错误映射，必须抛出异常由全局处理器统一转换。
4. 全局异常处理器中，系统异常必须记录完整堆栈日志（包含 `requestId`、请求路径、参数摘要）。
5. 全局异常处理器返回的响应结构必须与正常响应结构一致（参见 `common/api-design.md`）。

## 异常处理器示例

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusiness(BusinessException ex) {
        return ResponseEntity
            .status(ex.getHttpStatus())
            .body(ApiResponse.fail(ex.getErrorCode(), ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnknown(Exception ex) {
        log.error("未处理异常", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.fail("INTERNAL_ERROR", "服务内部错误"));
    }
}
```

## 异常传播（MUST）

1. Service 层捕获下游异常后，必须包装为自定义异常并保留根因（`new BusinessException(code, msg, cause)`）。
2. 禁止吞异常（空 catch 块），至少记录日志或重新抛出。
3. 系统异常必须记录日志（最少包含 `requestId`、操作、依赖标识、根因），但禁止原样透传给调用方。
4. `@Transactional` 方法中的异常必须注意回滚语义：默认仅 `RuntimeException` 回滚，checked exception 需显式配置 `rollbackFor`。
5. 禁止在多个 Controller 中重复手写异常映射逻辑，异常到响应的转换必须集中在 `@ControllerAdvice`。

## 错误码治理（MUST）

1. 同一服务内错误码唯一且语义稳定，必须使用枚举类集中定义，禁止在 Service 中内联硬编码错误码字符串。错误码枚举按模块拆文件（如 `UserErrorCode.java`、`OrderErrorCode.java`）。
2. 错误码格式推荐：`{模块}_{错误描述}`，如 `USER_NOT_FOUND`、`ORDER_DUPLICATE`。
3. 新增错误码必须补充文档和测试。
4. 业务错误码应按模块分段管理，避免不同模块错误码冲突。
5. 错误码枚举必须关联 HTTP 状态码，如 `USER_NOT_FOUND(404, "用户不存在")`。

### SHOULD
1. 错误码维护独立文档，前后端共享，便于联调。
2. 错误码变更必须在 CHANGELOG 中记录。
