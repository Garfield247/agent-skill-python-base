# 常量枚举、反模式避坑与静态诊断 (Constants, Anti-Patterns & Tooling)

## 1. 常量命名与枚举规范

- 常量采用大写蛇形（`UPPER_SNAKE_CASE`），配合 `typing.Final` 进行静态只读锁定：
  ```python
  from typing import Final
  DEFAULT_PAGE_SIZE: Final[int] = 20
  ```
- 状态与分类严格继承 `StrEnum` 或 `IntEnum`，严禁在业务逻辑中散落魔法字符串。

---

## 2. 核心反模式与避坑防线

### 2.1 🚨 默认可变参数死穴 (Mutable Default Argument)
- **绝对红线**：禁止在函数定义中使用 `items: list = []` 或 `d: dict = {}`，这会导致所有调用共享同一内存对象引发严重数据污染；
- **唯一正解**：默认值设为 `None`，在函数内部初始化。

### 2.2 🚨 生成器流式防 OOM (Generators over Full Lists)
- 大批量数据（海量文件行、数据库分页全量扫描）必须使用生成器表达式 `(x for x in ...)` 或 `yield` 流式处理，严禁一次性全部 `list(...)` 载入内存打爆进程。

### 2.3 🚨 异常链完整保留原则 (Exception Chaining)
- 捕获底层异常重抛业务异常时，**强制使用 `raise BaseBusinessException(...) from err`**，保留底层原始 Traceback。严禁裸 `except: pass` 静默吞掉错误。

---

## 3. 静态分析与排障工具链

```bash
# 1. 极速代码规范校验与自动修复 (Ruff)
ruff check . --fix
ruff format .

# 2. 严格模式静态类型检查 (Mypy)
mypy --strict .

# 3. 运行时内存分配追踪 (Tracemalloc)
python -m tracemalloc your_script.py
```
