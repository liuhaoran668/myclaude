# Python 项目架构分析模板

> 本模板用于系统化分析 Python 项目的架构设计、代码质量和潜在问题

---

## 第一阶段：宏观结构扫描

### 1.1 目录结构分析

```bash
# 快速了解项目结构
tree -L 3 -I '__pycache__|*.pyc|.git|venv|env' .
# 或
find . -name "*.py" -not -path "./.venv/*" | head -50
```

**检查项**：
- [ ] 是否有清晰的包结构？（`src/`, `tests/`, `docs/`）
- [ ] 是否存在 `__init__.py` 定义公共接口？
- [ ] 是否有 `setup.py` / `pyproject.toml` / `setup.cfg`？
- [ ] 依赖管理方式？（requirements.txt / poetry / pipenv）

### 1.2 入口点识别

**检查项**：
- [ ] `if __name__ == "__main__"` 位置
- [ ] `__init__.py` 中的 `__all__` 定义
- [ ] 命令行入口（console_scripts）
- [ ] 对外暴露的主类/函数

### 1.3 模块划分

绘制模块依赖图：

```
┌─────────────┐
│   用户代码   │
└──────┬──────┘
       │ import
┌──────▼──────┐
│  Facade 类   │  ← 如 LinkerHand, L6Hand
└──────┬──────┘
       │
┌──────▼──────┐
│  Manager 层  │  ← 如 AngleManager, TouchManager
└──────┬──────┘
       │
┌──────▼──────┐
│  通信/驱动层  │  ← 如 CANDriver, SerialDriver
└─────────────┘
```

---

## 第二阶段：设计模式识别

### 2.1 创建型模式

| 模式 | Python 识别特征 | 检查位置 |
|------|-----------------|----------|
| **单例** | `__new__` 覆写, 模块级实例, `@singleton` 装饰器 | 全局管理器、配置 |
| **工厂** | `create_xxx()`, `@classmethod` 创建方法 | 对象创建集中点 |
| **建造者** | 链式方法返回 `self`, `build()` 方法 | 复杂配置对象 |
| **原型** | `copy.copy()`, `copy.deepcopy()`, `__copy__` | 对象克隆场景 |

```python
# Python 单例实现方式
# 方式1：模块级单例（推荐）
# config.py
_config = None

def get_config():
    global _config
    if _config is None:
        _config = Config()
    return _config

# 方式2：类装饰器
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

# 方式3：元类
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
```

**检查项**：
- [ ] 单例是否线程安全？（使用 `threading.Lock`）
- [ ] 工厂方法是否支持扩展？

### 2.2 结构型模式

| 模式 | Python 识别特征 | 检查位置 |
|------|-----------------|----------|
| **Facade** | 组合多个子系统，提供简化接口 | 主入口类 |
| **Adapter** | 包装第三方库，转换接口 | 外部依赖封装 |
| **Decorator** | `@decorator` 语法，`functools.wraps` | 功能增强 |
| **Composite** | 树形结构，统一接口处理单个/组合 | 层级数据 |
| **Proxy** | `__getattr__` 转发，懒加载 | 远程调用、缓存 |

```python
# Python Facade 示例
class LinkerHand:
    """Facade：统一入口，隐藏内部复杂性"""

    def __init__(self, hand_type: str):
        # 内部组合多个子系统
        self._driver = CANDriver()
        self._angle_manager = AngleManager(self._driver)
        self._touch_manager = TouchManager(self._driver)
        self._force_manager = ForceSensorManager(self._driver)

    # 提供简化的高层接口
    def get_angle(self) -> List[float]:
        return self._angle_manager.get_current_angle()

    def set_angle(self, angles: List[float]):
        self._angle_manager.set_target_angle(angles)
```

**检查项**：
- [ ] Facade 是否暴露了内部 Manager？（如 `hand.angle_manager`）
- [ ] 暴露内部对象是否影响生命周期一致性？

### 2.3 行为型模式

| 模式 | Python 识别特征 | 检查位置 |
|------|-----------------|----------|
| **观察者** | `subscribe()`, `on_xxx()`, 回调列表 | 事件系统 |
| **策略** | 函数作为参数，协议类 | 算法切换 |
| **命令** | 命令对象/函数，支持撤销 | 操作队列 |
| **状态** | 状态类，`__class__` 切换 | 状态机 |
| **模板方法** | 抽象基类 + 钩子方法 | 算法框架 |

```python
# Python 观察者模式
class Observable:
    def __init__(self):
        self._callbacks: List[Callable] = []

    def subscribe(self, callback: Callable):
        self._callbacks.append(callback)
        return lambda: self._callbacks.remove(callback)  # 返回取消函数

    def notify(self, data):
        for cb in self._callbacks[:]:  # 复制列表防止迭代时修改
            cb(data)
```

### 2.4 Python 特有模式

| 模式 | 识别特征 | 典型应用 |
|------|----------|----------|
| **Mixin** | 多重继承，提供可复用功能 | `LoggingMixin`, `SerializableMixin` |
| **协议 (Protocol)** | `typing.Protocol`, duck typing | 结构化子类型 |
| **上下文管理器** | `__enter__`, `__exit__`, `@contextmanager` | 资源管理 |
| **描述符** | `__get__`, `__set__`, `__delete__` | 属性控制 |
| **元类** | `class Meta(type)` | 类创建控制 |

```python
# 上下文管理器示例
class HandConnection:
    def __enter__(self):
        self.connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False  # 不抑制异常

# 使用
with HandConnection() as hand:
    hand.set_angle([0, 0, 0, 0, 0, 0])
# 自动调用 close()
```

**检查项**：
- [ ] 是否利用上下文管理器管理资源？
- [ ] Mixin 类是否职责单一？
- [ ] 是否过度使用元类？（通常有更简单的替代方案）

---

## 第三阶段：生命周期契约分析

### 3.1 资源管理

**Python 资源管理方式**：

```python
# 方式1：上下文管理器（推荐）
with open("file.txt") as f:
    data = f.read()

# 方式2：显式 close
conn = Connection()
try:
    conn.do_something()
finally:
    conn.close()

# 方式3：atexit 注册
import atexit
atexit.register(cleanup_function)

# 方式4：__del__（不推荐，时机不确定）
class Resource:
    def __del__(self):
        self.close()  # 可能不被调用或调用时机不确定
```

**检查项**：
- [ ] 是否实现了 `__enter__` / `__exit__`？
- [ ] 是否依赖 `__del__` 释放资源？（危险）
- [ ] 是否有显式的 `close()` / `stop()` 方法？

### 3.2 状态管理

```python
from enum import Enum, auto

class State(Enum):
    CREATED = auto()
    RUNNING = auto()
    STOPPED = auto()
    CLOSED = auto()

class Hand:
    def __init__(self):
        self._state = State.CREATED

    def start(self):
        if self._state != State.CREATED:
            raise StateError(f"Cannot start from {self._state}")
        self._do_start()
        self._state = State.RUNNING

    def close(self):
        if self._state == State.CLOSED:
            return  # 幂等
        self._do_close()
        self._state = State.CLOSED
```

**检查项**：
- [ ] 是否有明确的状态枚举？
- [ ] 状态转换是否有验证？
- [ ] 非法操作是否抛出明确异常？

### 3.3 close 后行为

**检查项**：
- [ ] close 后调用各 API 的行为是否一致？
- [ ] 是否统一抛出特定异常（如 `StateError`）？
- [ ] 还是返回空值/默认值？

```python
class Hand:
    def _check_state(self):
        """统一状态检查，保证一致行为"""
        if self._state == State.CLOSED:
            raise StateError("Hand is closed")

    def get_angle(self):
        self._check_state()
        return self._angle_manager.get_angle()

    def set_angle(self, angles):
        self._check_state()
        self._angle_manager.set_angle(angles)
```

### 3.4 Facade 与 Manager 的关系

**问题场景**：
```python
hand = L6Hand()
manager = hand.angle_manager  # 暴露内部 Manager

hand.close()
manager.get_angle()  # 应该发生什么？
```

**检查项**：
- [ ] Facade 是否暴露内部 Manager？
- [ ] 暴露的 Manager 是否受 Facade 生命周期控制？
- [ ] 是否需要代理模式包装 Manager？

---

## 第四阶段：并发安全分析

### 4.1 GIL 的影响

**Python GIL（全局解释器锁）特点**：
- 同一时刻只有一个线程执行 Python 字节码
- I/O 操作会释放 GIL
- C 扩展可能释放 GIL

**检查项**：
- [ ] CPU 密集型任务是否使用多进程？
- [ ] I/O 密集型任务是否使用多线程/异步？
- [ ] 是否正确理解 GIL 不保护数据结构？

```python
# GIL 不保护的情况
counter = 0

def increment():
    global counter
    for _ in range(100000):
        counter += 1  # 非原子操作！

# 多线程下 counter 结果不确定
```

### 4.2 线程安全

```python
import threading

class ThreadSafeManager:
    def __init__(self):
        self._lock = threading.Lock()
        self._data = []

    def add(self, item):
        with self._lock:
            self._data.append(item)

    def get_all(self):
        with self._lock:
            return self._data.copy()  # 返回副本
```

**检查项**：
- [ ] 共享状态是否有锁保护？
- [ ] 锁的粒度是否合适？
- [ ] 是否存在死锁风险？

### 4.3 回调分析

```python
class DataSource:
    def __init__(self):
        self._callbacks = []
        self._lock = threading.Lock()

    def subscribe(self, callback):
        with self._lock:
            self._callbacks.append(callback)

    # 危险：在锁内调用回调
    def notify_unsafe(self, data):
        with self._lock:
            for cb in self._callbacks:
                cb(data)  # 回调可能阻塞或死锁

    # 安全：复制后在锁外调用
    def notify_safe(self, data):
        with self._lock:
            callbacks = self._callbacks.copy()
        for cb in callbacks:
            cb(data)

    # 更安全：使用线程池异步执行
    def notify_async(self, data):
        with self._lock:
            callbacks = self._callbacks.copy()
        for cb in callbacks:
            self._executor.submit(cb, data)
```

**检查项**：
- [ ] 回调在哪个线程执行？
- [ ] 回调是否可能阻塞？
- [ ] 回调内是否可以取消订阅？
- [ ] 对象销毁时回调如何处理？

### 4.4 异步编程（asyncio）

如果项目使用 asyncio：

**检查项**：
- [ ] 是否正确使用 `async/await`？
- [ ] 是否在异步函数中调用阻塞操作？
- [ ] 是否正确处理任务取消？
- [ ] 是否有未等待的协程警告？

```python
# 错误：在异步函数中阻塞
async def bad_example():
    time.sleep(1)  # 阻塞整个事件循环！

# 正确
async def good_example():
    await asyncio.sleep(1)

# 阻塞操作应在线程池执行
async def run_blocking():
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, blocking_function)
```

---

## 第五阶段：类型系统与接口设计

### 5.1 类型注解使用

**检查项**：
- [ ] 是否使用类型注解？
- [ ] 是否使用 `typing` 模块高级类型？
- [ ] 是否配置类型检查工具（mypy, pyright）？

```python
from typing import List, Optional, Callable, Protocol, TypeVar

class Driver(Protocol):
    """使用 Protocol 定义接口"""
    def send(self, data: bytes) -> bool: ...
    def receive(self) -> Optional[bytes]: ...

T = TypeVar('T')

class Manager(Generic[T]):
    def get_data(self) -> T: ...
```

### 5.2 抽象基类

```python
from abc import ABC, abstractmethod

class BaseManager(ABC):
    @abstractmethod
    def start(self) -> None:
        """启动管理器"""
        pass

    @abstractmethod
    def stop(self) -> None:
        """停止管理器"""
        pass

    def close(self) -> None:
        """关闭管理器（有默认实现）"""
        self.stop()
```

**检查项**：
- [ ] 是否使用 ABC 定义接口？
- [ ] 抽象方法是否有文档？
- [ ] 是否滥用抽象类？（Python 倾向鸭子类型）

### 5.3 数据类

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class HandConfig:
    hand_type: str
    can_interface: str = "can0"
    baud_rate: int = 1000000
    joint_count: int = 6

@dataclass
class SensorData:
    timestamp: float
    values: List[float] = field(default_factory=list)

    def __post_init__(self):
        if len(self.values) > 0 and not all(isinstance(v, float) for v in self.values):
            raise ValueError("All values must be float")
```

**检查项**：
- [ ] 是否使用 dataclass 简化数据类？
- [ ] 是否使用 `__post_init__` 进行验证？
- [ ] 不可变数据是否使用 `frozen=True`？

---

## 第六阶段：错误处理

### 6.1 异常层次

```python
# 定义项目异常层次
class HandError(Exception):
    """基础异常"""
    pass

class ConnectionError(HandError):
    """连接错误"""
    pass

class StateError(HandError):
    """状态错误"""
    pass

class TimeoutError(HandError):
    """超时错误"""
    pass

class ConfigError(HandError):
    """配置错误"""
    pass
```

**检查项**：
- [ ] 是否定义了项目异常基类？
- [ ] 异常是否有清晰的层次？
- [ ] 是否避免捕获 `Exception` / `BaseException`？

### 6.2 异常处理最佳实践

```python
# 好的实践
try:
    result = risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
    raise HandError("Operation failed") from e  # 异常链

# 避免
try:
    result = risky_operation()
except:  # 捕获所有异常，包括 KeyboardInterrupt
    pass  # 静默忽略
```

**检查项**：
- [ ] 是否正确使用异常链（`raise ... from ...`）？
- [ ] 是否有过于宽泛的异常捕获？
- [ ] 关键异常是否记录日志？

---

## 第七阶段：扩展性评估

### 7.1 开闭原则

```python
# 违反 OCP
def create_hand(hand_type: str):
    if hand_type == "L6":
        return L6Hand()
    elif hand_type == "L10":
        return L10Hand()
    # 每加一种手都要改

# 符合 OCP：注册机制
class HandRegistry:
    _registry: Dict[str, Type[Hand]] = {}

    @classmethod
    def register(cls, hand_type: str):
        def decorator(hand_class: Type[Hand]):
            cls._registry[hand_type] = hand_class
            return hand_class
        return decorator

    @classmethod
    def create(cls, hand_type: str, **kwargs) -> Hand:
        return cls._registry[hand_type](**kwargs)

@HandRegistry.register("L6")
class L6Hand(Hand):
    pass

@HandRegistry.register("L10")
class L10Hand(Hand):
    pass
```

### 7.2 依赖注入

```python
# 紧耦合
class Hand:
    def __init__(self):
        self._driver = CANDriver()  # 内部创建

# 松耦合
class Hand:
    def __init__(self, driver: Driver):  # 注入依赖
        self._driver = driver

# 使用
driver = CANDriver() if production else MockDriver()
hand = Hand(driver)
```

**检查项**：
- [ ] 依赖是注入还是内部创建？
- [ ] 是否可以替换依赖进行测试？

### 7.3 插件机制

```python
# 插件发现机制
import importlib.metadata

def discover_plugins():
    plugins = {}
    for ep in importlib.metadata.entry_points(group='myapp.plugins'):
        plugins[ep.name] = ep.load()
    return plugins
```

---

## 第八阶段：代码风格与规范

### 8.1 PEP 8 检查

**检查项**：
- [ ] 命名是否符合 PEP 8？
  - 类名：`PascalCase`
  - 函数/变量：`snake_case`
  - 常量：`UPPER_SNAKE_CASE`
  - 私有：`_leading_underscore`
- [ ] 缩进是否统一（4空格）？
- [ ] 行长度是否合理（79/120）？

### 8.2 文档规范

```python
def set_angle(self, angles: List[float], *, speed: float = 1.0) -> bool:
    """设置目标角度。

    Args:
        angles: 各关节目标角度列表，长度应与关节数一致。
        speed: 运动速度因子，范围 [0.1, 2.0]，默认 1.0。

    Returns:
        是否成功设置。

    Raises:
        ValueError: 角度列表长度不正确或速度超出范围。
        StateError: 设备未启动或已关闭。

    Example:
        >>> hand.set_angle([0.0, 30.0, 60.0, 90.0, 120.0, 150.0])
        True
    """
    ...
```

**检查项**：
- [ ] 公共 API 是否有文档字符串？
- [ ] 是否遵循 Google/NumPy/Sphinx 风格？
- [ ] 是否记录异常和示例？

### 8.3 工具配置

**检查项**：
- [ ] 是否配置 `pyproject.toml` / `setup.cfg`？
- [ ] 是否使用 linter（flake8, ruff）？
- [ ] 是否使用 formatter（black, yapf）？
- [ ] 是否使用类型检查（mypy, pyright）？
- [ ] 是否配置 pre-commit hooks？

---

## 第九阶段：输出架构文档

### 9.1 文档模板

```markdown
# [项目名] 架构分析报告

## 1. 项目概述
- 项目定位与核心功能
- 技术栈（Python 版本、主要依赖）
- 安装与使用方式

## 2. 架构分层
[分层图]

## 3. 核心模块
### 3.1 Facade 层
- 主入口类
- 对外 API 列表

### 3.2 Manager 层
- 各 Manager 职责
- Manager 间关系

### 3.3 驱动层
- 通信实现
- 硬件抽象

## 4. 设计模式清单
| 模式 | 应用位置 | 目的 |
|------|----------|------|

## 5. 生命周期契约
### 5.1 状态机
### 5.2 资源管理
### 5.3 close 后行为

## 6. 并发模型
### 6.1 线程模型
### 6.2 回调语义
### 6.3 GIL 影响

## 7. 类型系统
- 类型注解覆盖率
- 协议/抽象基类使用

## 8. 问题与建议
### 8.1 严重问题
### 8.2 改进建议

## 9. 附录
- 类图
- 序列图
```

---

## 快速检查清单

### 🔴 严重问题

- [ ] 资源泄露（未关闭文件/连接）
- [ ] 依赖 `__del__` 释放关键资源
- [ ] 无锁保护的共享可变状态
- [ ] 持锁调用用户回调
- [ ] 在异步函数中执行阻塞操作
- [ ] 过于宽泛的异常捕获（裸 `except`）

### 🟡 中等问题

- [ ] close 非幂等
- [ ] Facade 暴露内部对象且未控制生命周期
- [ ] 缺少类型注解
- [ ] 状态转换无验证
- [ ] 错误处理不一致

### 🟢 优化建议

- [ ] 可使用 dataclass 简化
- [ ] 可使用 Protocol 定义接口
- [ ] 可使用上下文管理器
- [ ] 文档字符串可更完善
- [ ] 可增加类型注解

---

## 分析流程总结

```
1. 目录结构 → 了解包布局
       ↓
2. __init__.py → 识别公共 API
       ↓
3. 类关系 → 绘制依赖图
       ↓
4. 设计模式 → 识别架构意图
       ↓
5. 生命周期 → 分析资源管理
       ↓
6. 并发模型 → 评估线程/异步安全
       ↓
7. 类型系统 → 检查接口定义
       ↓
8. 错误处理 → 评估健壮性
       ↓
9. 输出文档 → 形成分析报告
```

---

## Python 与 C++ 分析差异对照

| 维度 | Python | C++ |
|------|--------|-----|
| 资源管理 | 上下文管理器 / GC | RAII / 智能指针 |
| 多态 | 鸭子类型 / Protocol | 虚函数 / 模板 |
| 并发保护 | threading.Lock | std::mutex |
| GIL 影响 | 需考虑 | 无 |
| 静态类型 | 可选（type hints） | 强制 |
| 内存安全 | GC 管理 | 需手动保证 |
| 接口定义 | Protocol / ABC | 纯虚类 / Concepts |
| 异常安全 | 通常无需特别关注 | 需明确异常安全级别 |
