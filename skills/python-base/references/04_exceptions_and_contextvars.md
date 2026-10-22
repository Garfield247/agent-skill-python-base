# 业务异常体系与 ContextVars 异步上下文追踪 (Exceptions & ContextVars)

## 1. 统一业务异常基类体系 (`common/exceptions/base.py`)

严禁在业务逻辑中直接随手抛出裸 `raise Exception("msg")`，必须通过统一结构化异常体系进行抛转：

```python
# common/exceptions/base.py
from typing import Any

class BaseBusinessException(Exception):
    """全局统一业务异常基类 (拒绝对外暴露裸 500)"""

    def __init__(self, code: int, message: str, data: Any = None) -> None:
        super().__init__(message)
        self.code = code
        self.message = message
        self.data = data

    def to_dict(self) -> dict[str, Any]:
        return {"code": self.code, "msg": self.message, "data": self.data}
```

---

## 2. 异步与线程安全上下文管理 (`common/ctxdata/trace.py`)

严禁在异步高并发（FastAPI/Tornado/AsyncIO）中使用全局模块级变量存储请求私有数据。必须使用 Python 原生 `contextvars.ContextVar`：

```python
# common/ctxdata/trace.py
import contextvars
from typing import Final

_TRACE_ID_CTX: Final[contextvars.ContextVar[str]] = contextvars.ContextVar("trace_id", default="")
_USER_ID_CTX: Final[contextvars.ContextVar[int | None]] = contextvars.ContextVar("user_id", default=None)

def get_trace_id() -> str:
    return _TRACE_ID_CTX.get()

def set_trace_id(trace_id: str) -> contextvars.Token[str]:
    return _TRACE_ID_CTX.set(trace_id)

def get_current_user_id() -> int | None:
    return _USER_ID_CTX.get()

def set_current_user_id(user_id: int | None) -> contextvars.Token[int | None]:
    return _USER_ID_CTX.set(user_id)
```
