# Python 代码封装六大核心原则 (Encapsulation Principles)

## 1. 原则一：显式优于隐式，严禁滥用 `**kwargs` 穿透黑盒
- **反模式**：在业务逻辑、服务层或工具函数中滥用 `def process_data(*args, **kwargs): other_func(**kwargs)`，导致 IDE 类型补全失效并引发运行时 `KeyError`。
- **规范**：
  1. 公共 API 必须使用显式命名参数与类型注解；
  2. 善用**仅限关键字参数（Keyword-Only Arguments）** `*`，强制显式传参；
  3. 参数超过 4 个时，强制封装为强类型模型（Pydantic `BaseModel` 或 `@dataclass(slots=True)`）。

```python
# ✅ 仅限关键字参数
def search_users(*, keyword: str, page: int = 1, page_size: int = 20) -> list[User]:
    ...
```

---

## 2. 原则二：最小知识原则与显式对外暴露 (`__all__`)
- 模块级对外导出的符号必须显式定义 `__all__ = ["ExportA", "ExportB"]`，杜绝 `import *` 命名空间污染；
- 内部辅助函数、模块级单例、私有常量必须以单前导下划线 `_` 命名（如 `_helper()`）。

---

## 3. 原则三：防原始类型偏执 (Avoid Primitive Obsession)
- 杜绝在核心业务中滥用裸 `dict`、裸 `tuple` 或裸 `str`；
- 实体状态使用 `StrEnum`/`IntEnum` 约束；
- 领域 ID 区分使用 `typing.NewType`（如 `UserId = NewType("UserId", int)`）；
- 只读复合值对象使用 `@dataclass(frozen=True, slots=True)`。

---

## 4. 原则四：资源生命周期必须使用上下文管理器 (`with` / `async with`)
- 凡涉及文件句柄、网络会话、DB 连接、分布式锁或配置置换，**坚决禁止要求外部显式调用 `.close()` 或 `.release()`**；
- 必须实现 `__enter__/__exit__` 或使用 `@contextmanager` 确保异常时 100% 安全回收。

---

## 5. 原则五：内部自闭环并发/协程安全
- 涉及共享状态维护的类（单例、本地缓存、序列号生成器），其线程安全（`threading.Lock`）或异步协程安全（`asyncio.Lock`）必须在方法内部自闭环，严禁将锁暴露给调用者在外部加锁。

---

## 6. 原则六：拒绝透传空壳封装
- 严禁编写仅仅简单原样调用标准库或第三方库、没有任何校验、错误转换或业务增强的“洋葱皮空壳代码”。
