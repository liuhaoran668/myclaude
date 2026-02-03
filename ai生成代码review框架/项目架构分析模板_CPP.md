# C++ 项目架构分析模板

> 本模板用于系统化分析 C++ 项目的架构设计、代码质量和潜在问题

---

## 第一阶段：宏观结构扫描

### 1.1 目录结构分析

```bash
# 快速了解项目结构
tree -L 3 -d .
# 或
find . -type f -name "*.cpp" -o -name "*.h" -o -name "*.hpp" | head -50
```

**检查项**：
- [ ] 是否有清晰的分层？（如 `src/`, `include/`, `lib/`, `test/`）
- [ ] 头文件与实现文件是否分离？
- [ ] 是否存在 `CMakeLists.txt` 或其他构建配置？
- [ ] 第三方依赖如何管理？（vendor/, conan, vcpkg）

### 1.2 入口点识别

**检查项**：
- [ ] `main()` 函数位置
- [ ] 库的公共头文件入口（通常是单一 include 文件）
- [ ] 对外暴露的 API 是什么？

### 1.3 模块划分

绘制模块依赖图：

```
┌─────────────┐
│   用户代码   │
└──────┬──────┘
       │
┌──────▼──────┐
│   Facade层   │  ← 对外统一接口
└──────┬──────┘
       │
┌──────▼──────┐
│  Manager层   │  ← 业务逻辑
└──────┬──────┘
       │
┌──────▼──────┐
│  通信/驱动层  │  ← 底层实现
└─────────────┘
```

---

## 第二阶段：设计模式识别

### 2.1 创建型模式

| 模式 | 识别特征 | 检查位置 |
|------|----------|----------|
| **单例 (Singleton)** | `static getInstance()`, 私有构造函数 | 全局管理器、配置类 |
| **工厂 (Factory)** | `createXxx()`, `makeXxx()` 方法 | 对象创建集中点 |
| **建造者 (Builder)** | 链式调用 `.setA().setB().build()` | 复杂对象配置 |
| **原型 (Prototype)** | `clone()` 方法 | 对象复制场景 |

**检查项**：
- [ ] 单例是否线程安全？（Meyer's Singleton / call_once）
- [ ] 工厂是否支持扩展？（注册机制 vs 硬编码 switch）

### 2.2 结构型模式

| 模式 | 识别特征 | 检查位置 |
|------|----------|----------|
| **Facade** | 统一接口类，内部组合多个子系统 | API 入口类 |
| **Adapter** | 包装类，转换接口 | 第三方库封装 |
| **Decorator** | 包装类，增强功能，保持接口一致 | 功能扩展点 |
| **Composite** | 树形结构，统一处理单个/组合对象 | UI组件、命令树 |
| **Bridge** | 抽象与实现分离，两个独立继承体系 | 跨平台抽象 |

**检查项**：
- [ ] Facade 是否真正隐藏了内部复杂性？
- [ ] 是否存在"透传型" Facade（只是转发调用，没有增值）？

### 2.3 行为型模式

| 模式 | 识别特征 | 检查位置 |
|------|----------|----------|
| **观察者 (Observer)** | `subscribe()`, `notify()`, 回调列表 | 事件系统、数据订阅 |
| **策略 (Strategy)** | 接口类 + 多个实现，运行时切换 | 算法选择、协议切换 |
| **命令 (Command)** | 命令对象封装操作，支持撤销/队列 | 指令系统 |
| **状态 (State)** | 状态类封装行为，状态切换 | 有限状态机 |
| **模板方法 (Template Method)** | 基类定义骨架，子类实现钩子 | 算法框架 |

### 2.4 C++ 特有模式

| 模式 | 识别特征 | 典型应用 |
|------|----------|----------|
| **CRTP** | `class Derived : public Base<Derived>` | 静态多态、Mixin |
| **Pimpl** | 公共类持有 `Impl*` 指针 | ABI 稳定、编译隔离 |
| **RAII** | 构造获取资源，析构释放资源 | 锁、文件、连接 |
| **Type Erasure** | `std::function`, `std::any` | 类型无关容器 |
| **Policy-Based Design** | 模板参数传入策略类 | 高度可配置组件 |

**CRTP 深度检查**：
```cpp
// 典型 CRTP 结构
template<typename Derived>
class CRTPBase {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();  // 静态分发
    }
    void defaultImpl() { /* 默认行为 */ }
};

class Concrete : public CRTPBase<Concrete> {
public:
    void implementation() { /* 具体实现 */ }
};
```

**检查项**：
- [ ] CRTP 基类是否提供了合理的默认实现？
- [ ] 是否可以用普通虚函数替代？（性能 vs 可读性权衡）
- [ ] 继承层次是否过深？

---

## 第三阶段：生命周期契约分析

### 3.1 资源所有权

**检查项**：
- [ ] 使用 `unique_ptr` 表示独占所有权？
- [ ] 使用 `shared_ptr` 表示共享所有权？
- [ ] 是否存在裸指针？所有权语义是否明确？
- [ ] 是否存在循环引用？（shared_ptr + weak_ptr）

```cpp
// 所有权分析示例
class Manager {
    std::unique_ptr<Driver> driver_;      // ✓ 独占所有权，清晰
    std::shared_ptr<Cache> cache_;        // ? 为什么需要共享？
    Connection* conn_;                    // ✗ 裸指针，所有权不明
};
```

### 3.2 状态机定义

绘制对象生命周期状态图：

```
┌─────────┐   create()   ┌─────────┐   start()   ┌─────────┐
│ CREATED │ ───────────► │ READY   │ ──────────► │ RUNNING │
└─────────┘              └─────────┘             └────┬────┘
                                                      │
                              ┌───────────────────────┘
                              │ stop()
                              ▼
┌─────────┐   close()    ┌─────────┐
│ CLOSED  │ ◄─────────── │ STOPPED │
└─────────┘              └─────────┘
```

**检查项**：
- [ ] 每个状态下各 API 的行为是否明确？
- [ ] 非法状态转换是否抛异常或返回错误码？
- [ ] 状态查询是否线程安全？

### 3.3 关闭顺序

**检查项**：
- [ ] 是否先停生产者，再停消费者？
- [ ] 是否先停应用层，再停通信层？
- [ ] 析构函数是否调用 `close()`？是否安全？
- [ ] `close()` 是否幂等？（多次调用不出错）

```cpp
// 正确的关闭顺序示例
class Hand {
public:
    ~Hand() {
        close();  // 析构时自动关闭
    }

    void close() {
        if (closed_.exchange(true)) return;  // 幂等

        // 1. 先停止上层订阅
        callbacks_.clear();

        // 2. 停止工作线程
        worker_.stop();

        // 3. 最后关闭底层连接
        driver_->close();
    }

private:
    std::atomic<bool> closed_{false};
};
```

### 3.4 close 后行为矩阵

| API | RUNNING | STOPPED | CLOSED |
|-----|---------|---------|--------|
| `getData()` | 返回数据 | 返回缓存/空 | 抛异常/返回错误 |
| `sendCommand()` | 执行 | 排队/拒绝 | 抛异常/返回错误 |
| `subscribe()` | 注册 | 注册（延迟生效） | 抛异常/返回错误 |
| `close()` | 执行关闭 | 执行关闭 | 无操作（幂等） |

---

## 第四阶段：并发安全分析

### 4.1 共享状态识别

**检查项**：
- [ ] 列出所有成员变量
- [ ] 标记哪些可能被多线程访问
- [ ] 确认每个共享状态的保护机制

```cpp
class DataManager {
    // 只读（构造后不变）- 无需保护
    const Config config_;

    // 单线程访问 - 无需保护
    InternalState internal_;

    // 多线程访问 - 需要保护
    std::mutex mutex_;
    std::vector<Data> buffer_;  // 受 mutex_ 保护

    // 原子变量 - 自带保护
    std::atomic<bool> running_{false};
    std::atomic<size_t> count_{0};
};
```

### 4.2 锁分析

**检查项**：
- [ ] 锁粒度是否合适？（太粗影响性能，太细容易出错）
- [ ] 是否存在锁顺序不一致？（死锁风险）
- [ ] 是否持锁调用回调？（用户代码可能再次加锁）
- [ ] 条件变量是否配合正确谓词？

```cpp
// 死锁风险示例
class A {
    void foo() {
        std::lock_guard<std::mutex> lock(mutex_a_);
        b_->bar();  // bar() 内部锁 mutex_b_，再调用 a_->xxx() 锁 mutex_a_ → 死锁
    }
};

// 修复：统一锁顺序，或使用 std::scoped_lock
```

### 4.3 回调安全

**核心问题**：回调在哪个线程执行？

```cpp
// 危险：在锁内调用回调
void notify() {
    std::lock_guard<std::mutex> lock(mutex_);
    for (auto& cb : callbacks_) {
        cb(data_);  // ✗ 用户回调可能阻塞或再次加锁
    }
}

// 安全：复制后在锁外调用
void notify() {
    std::vector<Callback> cbs;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        cbs = callbacks_;  // 复制
    }
    for (auto& cb : cbs) {
        cb(data_);  // ✓ 锁外调用
    }
}

// 更安全：异步派发
void notify() {
    auto data = getData();
    for (auto& cb : callbacks_) {
        executor_.post([cb, data] { cb(data); });  // ✓ 异步执行
    }
}
```

**检查项**：
- [ ] 回调是同步还是异步执行？
- [ ] 回调内是否允许调用 `unsubscribe()`？
- [ ] 对象销毁时是否等待所有 in-flight 回调完成？

### 4.4 停止与并发

**检查项**：
- [ ] `stop()` 是否能中断阻塞操作？
- [ ] `stop()` 后新请求如何处理？
- [ ] 是否存在 stop 与 send 的竞态？

```cpp
// 推荐的停止模式
class Worker {
public:
    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopping_ = true;
        }
        cond_.notify_all();  // 唤醒等待者

        if (thread_.joinable()) {
            thread_.join();  // 等待线程结束
        }
    }

private:
    void run() {
        while (true) {
            std::unique_lock<std::mutex> lock(mutex_);
            cond_.wait(lock, [this] {
                return stopping_ || !queue_.empty();
            });

            if (stopping_ && queue_.empty()) break;

            // 处理任务...
        }
    }
};
```

---

## 第五阶段：扩展性评估

### 5.1 开闭原则检查

**检查项**：
- [ ] 新增功能是否需要修改现有代码？
- [ ] 是否存在大量 `if-else` 或 `switch` 分支？
- [ ] 是否可以通过注册机制扩展？

```cpp
// 违反 OCP
void process(DeviceType type) {
    switch (type) {
        case L6: handleL6(); break;
        case L10: handleL10(); break;
        // 每加一种设备都要改这里
    }
}

// 符合 OCP
class DeviceRegistry {
    std::map<DeviceType, std::unique_ptr<Handler>> handlers_;
public:
    void registerHandler(DeviceType type, std::unique_ptr<Handler> h) {
        handlers_[type] = std::move(h);
    }
    void process(DeviceType type) {
        handlers_.at(type)->handle();  // 自动扩展
    }
};
```

### 5.2 依赖注入检查

**检查项**：
- [ ] 依赖是通过构造函数注入还是内部创建？
- [ ] 是否可以替换依赖进行测试？
- [ ] 接口是否足够抽象？

```cpp
// 紧耦合（难测试）
class Hand {
    CANDriver driver_;  // 内部创建，无法替换
public:
    Hand() : driver_("can0") {}
};

// 松耦合（可测试）
class Hand {
    std::unique_ptr<IDriver> driver_;  // 接口依赖
public:
    Hand(std::unique_ptr<IDriver> driver) : driver_(std::move(driver)) {}
};

// 测试时
auto mockDriver = std::make_unique<MockDriver>();
Hand hand(std::move(mockDriver));
```

### 5.3 模块边界清晰度

**检查项**：
- [ ] 模块间依赖是否单向？
- [ ] 是否存在循环依赖？
- [ ] 公共接口是否最小化？

---

## 第六阶段：代码风格与规范

### 6.1 命名规范

| 元素 | 常见风格 | 示例 |
|------|----------|------|
| 类名 | PascalCase | `HandController` |
| 函数名 | camelCase / snake_case | `getData()` / `get_data()` |
| 成员变量 | 带后缀下划线 | `data_`, `mutex_` |
| 常量 | 全大写 + 下划线 | `MAX_RETRY_COUNT` |
| 命名空间 | 小写 | `linker::hand` |

**检查项**：
- [ ] 项目内命名风格是否一致？
- [ ] 是否遵循团队/公司规范？

### 6.2 现代 C++ 特性使用

**检查项**：
- [ ] 是否使用 `auto` 简化类型声明？
- [ ] 是否使用范围 for 循环？
- [ ] 是否使用智能指针替代裸指针？
- [ ] 是否使用 `constexpr` 进行编译期计算？
- [ ] 是否使用 `std::optional` 表示可选值？
- [ ] 是否使用 `std::variant` 替代 union？

### 6.3 错误处理

**检查项**：
- [ ] 使用异常还是错误码？是否一致？
- [ ] 异常安全级别是否满足需求？
- [ ] 析构函数是否 noexcept？
- [ ] 是否有清晰的错误分类？

```cpp
// 错误分类示例
enum class ErrorCode {
    // 可恢复错误
    Timeout,
    Busy,

    // 不可恢复错误
    InvalidState,
    HardwareFailure,

    // 编程错误
    InvalidArgument,
    NullPointer,
};
```

---

## 第七阶段：输出架构文档

### 7.1 文档模板

```markdown
# [项目名] 架构分析报告

## 1. 项目概述
- 项目定位
- 核心功能
- 技术栈

## 2. 架构分层
[绘制分层图]

## 3. 核心模块
### 3.1 模块A
- 职责
- 对外接口
- 依赖关系
- 设计模式

## 4. 设计模式清单
| 模式 | 应用位置 | 目的 |
|------|----------|------|
| ... | ... | ... |

## 5. 生命周期契约
### 5.1 状态机
### 5.2 关闭顺序
### 5.3 API 行为矩阵

## 6. 并发模型
### 6.1 线程模型
### 6.2 共享状态保护
### 6.3 回调语义

## 7. 问题与建议
### 7.1 严重问题
### 7.2 改进建议
### 7.3 扩展性评估

## 8. 附录
- 类图
- 序列图
- 关键代码片段
```

---

## 快速检查清单

### 🔴 严重问题（必须修复）

- [ ] 裸指针所有权不明
- [ ] 数据竞争（无锁保护的共享状态）
- [ ] 持锁调用用户回调
- [ ] 死锁风险（锁顺序不一致）
- [ ] 资源泄露（未正确释放）
- [ ] 析构顺序问题

### 🟡 中等问题（建议修复）

- [ ] close 非幂等
- [ ] 状态转换边界不清
- [ ] 错误处理不一致
- [ ] 过度使用继承
- [ ] 循环依赖

### 🟢 优化建议

- [ ] 可使用更合适的设计模式
- [ ] 可提取公共逻辑
- [ ] 命名可更清晰
- [ ] 可增加 constexpr/noexcept
- [ ] 文档可更完善

---

## 分析流程总结

```
1. 目录结构 → 了解项目布局
       ↓
2. 入口点 → 找到 API 表面
       ↓
3. 类关系 → 绘制依赖图
       ↓
4. 设计模式 → 识别架构意图
       ↓
5. 生命周期 → 分析资源管理
       ↓
6. 并发模型 → 评估线程安全
       ↓
7. 扩展性 → 评估可维护性
       ↓
8. 输出文档 → 形成分析报告
```
