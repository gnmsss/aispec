# rules/python-server/common/error-handling.md

## 异常分类
1. 异常必须区分业务异常与系统异常，定义自定义异常基类。
2. 业务异常（`AppException`）携带错误码和可控消息，直接映射为客户端响应。
3. 系统异常（`SystemException`）携带依赖标识和操作上下文，仅记录日志，不透传给调用方。
4. 用户可见错误信息必须可控，禁止泄露 SQL、密钥、内部路径、堆栈信息。

### 异常基类示例
```python
class AppException(Exception):
    """业务异常基类，携带错误码和可控消息。"""

    def __init__(self, code: str, message: str, status_code: int = 400):
        self.code = code
        self.message = message
        self.status_code = status_code
        super().__init__(message)

class SystemException(Exception):
    """系统异常基类，记录日志但不透传给调用方。"""

    def __init__(self, message: str, cause: Exception | None = None):
        self.cause = cause
        super().__init__(message)
```

## 异常归属与目录
1. 通用异常机制（基类、错误码类型、异常处理器）放在 `app/core/exceptions/` 或 `app/exceptions/`，不放业务语义异常。
2. 带业务语义的异常必须放在各自模块目录（如 `app/modules/user/exceptions.py`、`app/modules/order/exceptions.py`）。
3. 系统异常仅用于系统级分类或封装，不得直接作为对外响应内容返回。

## 全局异常处理器（MUST）
1. FastAPI 项目 MUST 注册 `exception_handler`，统一捕获 `AppException`、`ValidationError`、`Exception`。
2. Django 项目 MUST 使用 DRF 的 `EXCEPTION_HANDLER` 自定义异常处理。
3. Flask 项目 MUST 注册 `@app.errorhandler` 统一处理。
4. 异常处理器必须将异常映射为统一响应结构（`code`、`message`、`data`、`request_id`、`timestamp`）。
5. 禁止在多个路由中重复手写 `try/except` 映射逻辑，异常到响应的转换必须集中治理。

### FastAPI 全局异常处理器示例
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "code": exc.code,
            "message": exc.message,
            "data": None,
            "request_id": request.state.request_id,
            "timestamp": datetime.now(UTC).isoformat(),
        },
    )

@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.error("unhandled_exception", exc_info=exc, request_id=request.state.request_id)
    return JSONResponse(
        status_code=500,
        content={
            "code": "INTERNAL_ERROR",
            "message": "服务内部错误",
            "data": None,
            "request_id": request.state.request_id,
            "timestamp": datetime.now(UTC).isoformat(),
        },
    )
```

## 异常传播
1. 禁止吞异常；捕获异常时必须记录或重新抛出，禁止空 `except: pass`。
2. 系统异常必须记录日志（最少包含 `request_id`、操作、依赖标识、根因），但禁止原样透传给调用方。
3. 边界层（HTTP/gRPC/消息）必须通过统一异常处理器将内部异常映射成稳定业务错误码与可控消息。
4. 异常链必须保留根因：使用 `raise AppException(...) from original_exception`。
5. 禁止使用裸 `except:`，必须指定具体异常类型；确需捕获所有异常时使用 `except Exception:`。

## 错误码治理（MUST）
1. 同一服务内错误码唯一且语义稳定，必须定义为常量或枚举，按模块集中管理（如 `app/modules/user/error_codes.py`、`app/modules/order/error_codes.py`），禁止在 service 中内联硬编码错误码字符串。
2. 业务异常必须引用集中定义的错误码创建（如 `raise AppException(UserErrors.NOT_FOUND, "用户不存在")`），禁止 `raise AppException("随意编码", "随意消息")`。
3. 新增错误码必须补充文档和测试。
4. 业务错误码应按作用域分段管理（如 `USER_` 前缀、`ORDER_` 前缀），避免不同模块错误码冲突。
5. 错误码推荐使用字符串而非数字（如 `"USER_NOT_FOUND"` 而非 `40001`），提高可读性。

## 上下文传播
1. 异常中必须携带足够的上下文信息，便于日志排查：操作名称、资源标识、依赖名称。
2. 异步任务（Celery）中的异常必须捕获并记录到日志和监控，禁止静默丢失。
3. 后台任务异常不影响主请求链路，但必须有独立的错误通知和重试机制。
