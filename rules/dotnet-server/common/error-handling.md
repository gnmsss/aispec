# rules/dotnet-server/common/error-handling.md

## 异常分类
1. 异常必须区分业务异常与系统异常，调用方可通过异常类型判断。
2. 业务异常（如用户不存在、余额不足）使用自定义异常类型，携带业务错误码和可展示消息。
3. 系统异常（如数据库连接失败、外部 API 超时）必须带上依赖标识和操作上下文，便于定位。
4. 用户可见错误信息必须可控，禁止泄露 SQL、堆栈、密钥、内部路径。

## 异常类型设计
1. 推荐定义基类 `BusinessException`（携带 `ErrorCode` 和 `Message`），各业务域继承扩展。
2. 示例：
   ```csharp
   /// <summary>
   /// 业务异常基类
   /// </summary>
   public class BusinessException : Exception
   {
       public string ErrorCode { get; }

       public BusinessException(string errorCode, string message)
           : base(message)
       {
           ErrorCode = errorCode;
       }
   }
   ```
3. 系统异常不需要自定义类型，直接使用框架内置异常（`InvalidOperationException`、`TimeoutException` 等），在异常处理中间件中统一捕获。

## 异常归属与目录
1. 通用异常基类和错误码类型放在共享项目（如 `*.Shared` 或 `*.Domain`），不放业务语义异常。
2. 带业务语义的异常必须放在对应业务模块目录，并按作用域拆文件（例如 `UserException.cs`、`OrderException.cs`）。
3. 推荐将错误码定义为常量类，按业务域分文件管理（如 `UserErrorCodes.cs`、`OrderErrorCodes.cs`）。

## 异常传播
1. 禁止吞异常；捕获异常后必须处理（记录日志、转换抛出、或明确忽略并注释原因）。
2. 重新抛出异常时必须保留原始堆栈：使用 `throw` 而非 `throw ex`。
3. 系统异常必须记录日志（最少包含 `RequestId`、操作、依赖标识、根因），但禁止原样透传给调用方。
4. 边界层（HTTP/gRPC/消息）必须通过统一异常处理中间件将内部异常映射成稳定业务错误码与可控消息。
5. 禁止在多个 Controller 中重复手写异常映射逻辑，异常到响应的转换必须集中治理。

## 统一异常处理中间件
1. 推荐使用 `IExceptionHandler`（.NET 8+）或自定义 ExceptionHandling Middleware 统一处理。
2. 中间件职责：
   - `BusinessException` → 提取 `ErrorCode` 和 `Message`，返回 `4xx` + 业务响应。
   - 未捕获系统异常 → 记录完整日志，返回 `500` + 通用错误响应（不泄露细节）。
   - 参数校验异常 → 返回 `400` + 字段级错误详情。
3. 示例：
   ```csharp
   public class GlobalExceptionHandler : IExceptionHandler
   {
       private readonly ILogger<GlobalExceptionHandler> _logger;

       public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
       {
           _logger = logger;
       }

       public async ValueTask<bool> TryHandleAsync(
           HttpContext context, Exception exception, CancellationToken ct)
       {
           switch (exception)
           {
               case BusinessException biz:
                   context.Response.StatusCode = 400;
                   await context.Response.WriteAsJsonAsync(new
                   {
                       code = biz.ErrorCode,
                       message = biz.Message,
                       requestId = context.TraceIdentifier,
                       timestamp = DateTime.UtcNow
                   }, ct);
                   return true;

               default:
                   _logger.LogError(exception,
                       "未处理异常 RequestId={RequestId}", context.TraceIdentifier);
                   context.Response.StatusCode = 500;
                   await context.Response.WriteAsJsonAsync(new
                   {
                       code = "INTERNAL_ERROR",
                       message = "服务器内部错误",
                       requestId = context.TraceIdentifier,
                       timestamp = DateTime.UtcNow
                   }, ct);
                   return true;
           }
       }
   }
   ```

## 错误码治理（MUST）
1. 同一服务内错误码唯一且语义稳定，必须定义为常量类，按业务域分文件管理（如 `UserErrorCodes.cs`、`OrderErrorCodes.cs`），禁止在 Service 中内联硬编码错误码字符串。
2. 业务异常必须引用集中定义的错误码常量构造（如 `throw new BusinessException(UserErrorCodes.NotFound, "用户不存在")`），禁止 `throw new BusinessException("随意编码", "随意消息")`。
3. 新增错误码必须补充文档和测试。
4. 业务错误码应按作用域分段管理，避免 `User`、`Order`、`System` 等作用域互相混用或冲突。
