# common/ 基础层分层架构标准与 *x 扩展工具包 (Common Architecture & Helpers)

## 1. common/ 一级目录收紧原则 (收紧为 6~7 个核心基础大类)

```text
common/                       # 基础公共层根目录
├── enums/                    # [核心 1] 全局跨域业务枚举定义中心
├── types/                    # [核心 2] 跨域纯类型契约、泛型与值对象 (Pydantic / Dataclass)
├── exceptions/               # [核心 3] 全局统一业务异常体系与错误码
├── ctxdata/                  # [核心 4] 全异步/多线程安全上下文 (contextvars.ContextVar)
├── utils/                    # [核心 5] 通用无状态纯计算工具集 (方案 B: *x 扩展模块)
│   ├── listx.py              # 切片/列表/迭代器高级扩展 (chunk, unique, flatten)
│   ├── dictx.py              # 字典/深层结构高级扩展 (get_in, deep_merge, pick, omit)
│   ├── strx.py               # 字符串安全截断与敏感信息脱敏 (safe_truncate, mask_phone)
│   ├── timex.py              # 统一时区感知与自然日对齐 (SHANGHAI_TZ, now_shanghai)
│   └── convx.py              # 安全类型转换与兜底降级 (to_int, to_float, to_bool)
├── storage/                  # [核心 6] 外部基础持久化与缓存客户端连接封装
└── middlewares/              # [核心 7] Web/RPC 通用拦截器与中间件
```

> **命名红线**：严禁采用多单词无间隔挤压拼接（禁止 `maputil.py`, `stringutil.py`）。统一采用 **方案 B：`*x` 扩展命名法**（`listx.py`, `dictx.py`, `strx.py`, `timex.py`, `convx.py`），彻底规避与 Python 标准库同名模块发生遮蔽。

---

## 2. 核心工具包生产级实现范例

### 2.1 `listx.py`：切片流式分块与保序去重
```python
from collections.abc import Callable, Hashable, Iterable, Iterator
from itertools import islice
from typing import TypeVar

T = TypeVar("T")
K = TypeVar("K", bound=Hashable)

def chunk(iterable: Iterable[T], size: int) -> Iterator[list[T]]:
    """流式分块迭代器 (防 OOM)，将可迭代对象按指定大小切分"""
    if size <= 0:
        raise ValueError("size 必须大于 0")
    iterator = iter(iterable)
    while batch := list(islice(iterator, size)):
        yield batch

def unique(iterable: Iterable[T], key: Callable[[T], K] | None = None) -> list[T]:
    """保序去重：保持原序列元素初次出现的相对顺序"""
    seen: set[Hashable] = set()
    result: list[T] = []
    for item in iterable:
        val = key(item) if key else item
        if val not in seen:
            seen.add(val)
            result.append(item)
    return result
```

### 2.2 `dictx.py`：安全深层嵌套取值
```python
from collections.abc import Mapping
from typing import Any

def get_in(d: Mapping[str, Any], path: Iterable[str], default: Any = None) -> Any:
    """安全深层嵌套取值，防止 KeyError 或 TypeError 崩溃
    用法: get_in(user_dict, ["profile", "address", "city"], default="未知")
    """
    curr: Any = d
    for key in path:
        if isinstance(curr, Mapping) and key in curr:
            curr = curr[key]
        else:
            return default
    return curr
```

### 2.3 `timex.py`：时区感知与标准时间格式常量
```python
from datetime import datetime, time, timezone
from typing import Final
from zoneinfo import ZoneInfo

DATETIME_FORMAT: Final[str] = "%Y-%m-%d %H:%M:%S"
DATE_FORMAT: Final[str] = "%Y-%m-%d"
SHANGHAI_TZ: Final[ZoneInfo] = ZoneInfo("Asia/Shanghai")
UTC_TZ: Final[timezone] = timezone.utc

def now_shanghai() -> datetime:
    """获取当前上海时区的感知时间 (Timezone-Aware)"""
    return datetime.now(SHANGHAI_TZ)
```
