# C++系统编程设计原则

实战导向的设计决策指引。每条原则包含: 信号(何时触发) + 机制(怎么做) + C++代码示例。
按使用场景组织: 设计基石 → 编码前审视 → 编码中遵循 → 编码后回看。

---

## 一、设计基石

内化于心的原则，贯穿设计、编码、审查全过程。

### 1. SOLID原则

SOLID不是5个独立原则，而是从不同视角描述同一件事的整体:

```
SRP: 按变化原因分离 --+
                       +-- 高内聚 --+
ISP: 按客户端需求分离 --+           |
                                    +-- 可维护、可扩展的代码
OCP: 扩展不改旧代码 --+            |
                       +-- 低耦合 --+
DIP: 依赖抽象不依赖实现 --+

LSP: 抽象正确 -- 是上述一切的基础
```

#### SRP — 单一职责

本质: 一个模块只对一个actor(利益相关者)负责。"改变的理由"指的是人，不是技术原因。

信号 — 代码违反SRP:
- 构造函数参数 >5个
- 一个类同时处理序列化和业务逻辑
- 一个函数既做计算又做I/O
- 修改一个功能时不相关的测试失败
- 类名含"Manager"、"Handler"、"Processor"、"Utils"

机制:
1. 识别actor: 这段代码服务于哪些不同的利益相关者?
2. 按actor划分: 因同一actor变化的代码聚合在一起
3. 分离变化轴: 因不同原因变化的逻辑拆到不同类
4. Facade聚合: 拆分后对外用Facade提供统一入口

```cpp
// 违反SRP: 三个变化原因混在一起
class Connection {
    void connect(const Endpoint& ep);
    void disconnect();
    Buffer serialize(const Message& msg);    // 协议变化
    Message deserialize(const Buffer& buf);
    void sendWithRetry(const Message& msg);  // 重试策略变化
};

// 遵循SRP: 按变化原因拆分，Facade组合
class Transport { /* 网络连接 */ };
class Codec     { /* 编解码 */ };
class RetryPolicy {
    template<typename F>
    auto execute(F&& fn) -> decltype(fn());
};

class Connection {  // Facade
    Transport transport_;
    Codec codec_;
    RetryPolicy retry_;
public:
    void send(const Message& msg) {
        auto buf = codec_.encode(msg);
        retry_.execute([&]{ transport_.send(buf); });
    }
};
```

陷阱:
- 过度拆分: 3行代码且不会独立变化的职责不值得独立成类
- 过早应用: 第一次编写时不预测所有actor，当观察到反复因不同原因变化时再拆
- Parnas洞察: 模块划分不应基于流程图，而应基于"可能变化的设计决策"

#### OCP — 开放封闭

本质: 新功能通过编写新代码添加，不修改旧代码。

信号 — 代码违反OCP:
- switch/if-else根据类型做不同处理，且每次新增类型都要改
- 直接依赖具体类，新增实现时必须改调用方
- 霰弹枪手术: 添加新功能需在多个文件修改

六种C++实现手段:

| 手段 | 特点 | 适用场景 |
|------|------|---------|
| virtual继承 | 运行时绑定 | 类型可变，操作固定 |
| 模板 | 编译时绑定，零开销 | 性能极敏感 |
| CRTP | 编译期多态 | 需要基类调用派生类方法 |
| std::function | 灵活注入行为 | 轻量回调 |
| 模板方法 | 骨架不变，步骤可变 | 流程统一 |
| variant+visit | 类型固定，操作可变 | 与virtual方向相反 |

```cpp
// 违反OCP: 每次新增算法都要改switch
void allReduce(void* buf, size_t count, AlgoType algo) {
    switch (algo) {
        case AlgoType::Ring:        ringAllReduce(buf, count); break;
        case AlgoType::RecursiveHD: recursiveHDAllReduce(buf, count); break;
    }
}

// 遵循OCP: 新增算法只写新代码
class AllReduceAlgo {
public:
    virtual ~AllReduceAlgo() = default;
    virtual void execute(void* buf, size_t count, Communicator& comm) = 0;
};
class RingAllReduce : public AllReduceAlgo { /* ... */ };

// 新增算法: 只添加新类
class PairwiseExchange : public AllReduceAlgo { /* ... */ };
```

陷阱:
- 完美OCP不存在 — OCP是重构方向，不是初始设计目标
- 不是每个依赖点都需要抽象，判断标准: 这个点未来是否真的有多个实现?
- OCP有两个根本限制: (1)仍需注册/切换机制 (2)必须提前设计扩展方向(新类型 vs 新操作)

#### LSP — 里氏替换

本质: 子类型必须能替换基类型而不改变程序正确性。

信号 — 代码违反LSP:
- 多态代码中出现`dynamic_cast`或`typeid`
- 基类出现"类型标识位"区分子类
- 子类重写方法时抛出基类未声明的异常
- 子类前置条件比基类更严格

四种违反模式:
1. 前置条件加强: 子类对输入施加更严格约束
2. 后置条件削弱: 子类返回结果不满足基类承诺
3. 不变量破坏: 子类引入破坏基类不变量的状态修改
4. 历史规则违反: 子类引入基类不存在的状态变迁

```cpp
// 违反LSP: SharedMemTransport的send有基类未声明的size限制
class Transport {
public:
    virtual void send(const void* buf, size_t len) = 0;
};
class SharedMemTransport : public Transport {
    void send(const void* buf, size_t len) override {
        if (len > shm_size_) throw std::runtime_error("too large");
        // 前置条件更严格 — 违反LSP
    }
};

// 遵循LSP: 约束成为契约的一部分
class Transport {
public:
    virtual size_t maxMessageSize() const = 0;  // 在接口层声明约束
    virtual void send(const void* buf, size_t len) = 0;
};
// 调用方统一处理，无需类型检查
void broadcast(Transport& t, const void* data, size_t len) {
    if (len > t.maxMessageSize()) { /* 分块发送 */ }
    t.send(data, len);
}
```

检验方法: 为基类编写黑盒测试，所有子类都必须通过。用"IS-SUBSTITUTABLE-FOR"替代"IS-A"判断继承关系。

#### ISP — 接口隔离

本质: 客户端不应被迫依赖用不到的方法。

信号: 实现类中出现空方法体或`throw "not implemented"`。

```cpp
// 违反ISP: TCP设备被迫实现RDMA方法
class IDevice {
    virtual void send(const Buffer& buf) = 0;
    virtual void rdmaWrite(const MemRegion& l, const MemRegion& r) = 0;
    virtual void atomicAdd(uint64_t* remote, uint64_t val) = 0;
};
class TcpDevice : public IDevice {
    void rdmaWrite(...) override { throw std::runtime_error("not supported"); }
};

// 遵循ISP: 按能力拆分
class ISendRecv { virtual void send(const Buffer&) = 0; };
class IRdma     { virtual void rdmaWrite(const MemRegion&, const MemRegion&) = 0; };
class IAtomic   { virtual void atomicAdd(uint64_t*, uint64_t) = 0; };

class TcpDevice  : public ISendRecv { /* 只实现收发 */ };
class RdmaDevice : public ISendRecv, public IRdma, public IAtomic { /* 全部 */ };
```

ISP的实际效果: (1)编译速度 — fat interface变更触发全量重编译 (2)可读性 — `MyDevice : ISendRecv, IRdma`一目了然 (3)消除意外耦合。

#### DIP — 依赖反转

本质: 高层不依赖低层，两者都依赖抽象。

信号: 高层逻辑中直接`#include`低层实现头文件，`new ConcreteClass`散布在业务逻辑中。

四种C++实现方式:

| 方式 | vtable开销 | 内联优化 | 解耦程度 | 适用场景 |
|------|-----------|---------|---------|---------|
| 虚函数 | 有 | 不可 | 中 | 通用，类型数适中 |
| 模板 | 无 | 可 | 低(编译期) | 性能极敏感 |
| std::function | 有(SOO可避免) | 不可 | 中 | 轻量回调 |
| Type Erasure | 有 | 不可 | 高 | 具体类不应知道接口 |

```cpp
// 遵循DIP — 虚函数注入
class ITransport {
public:
    virtual void send(const void* buf, size_t len) = 0;
    virtual ~ITransport() = default;
};
class CollectiveScheduler {
    ITransport& transport_;
public:
    explicit CollectiveScheduler(ITransport& t) : transport_(t) {}
    void allReduce(void* buf, size_t count) {
        transport_.send(buf, count);
    }
};

// 遵循DIP — 模板注入(零开销)
template<typename Transport>
class CollectiveScheduler {
    Transport transport_;
public:
    void allReduce(void* buf, size_t count) {
        transport_.send(buf, count);  // 编译时绑定，可内联
    }
};

// 遵循DIP — Type Erasure(具体类无需知道抽象接口)
class AnyTransport {
    struct Concept {
        virtual void send(const void*, size_t) = 0;
        virtual ~Concept() = default;
    };
    template<typename T>
    struct Model : Concept {
        T impl_;
        explicit Model(T impl) : impl_(std::move(impl)) {}
        void send(const void* buf, size_t len) override { impl_.send(buf, len); }
    };
    std::unique_ptr<Concept> ptr_;
public:
    template<typename T>
    AnyTransport(T impl) : ptr_(std::make_unique<Model<T>>(std::move(impl))) {}
    void send(const void* buf, size_t len) { ptr_->send(buf, len); }
};
// RdmaTransport不需要继承任何接口 — 真正的解耦
struct RdmaTransport { void send(const void* buf, size_t len); };
AnyTransport t = RdmaTransport{};
```

YAGNI原则: 虚函数够用就不要上type erasure。先用最简单方式，复杂度不够时再升级。

DIP/IoC/DI三者关系: DIP是设计原则(目的)，IoC是设计模式(思路: 框架调用用户代码)，DI是实现手段(通过构造函数/setter/模板参数注入实现)。

### 2. 信息隐藏与模块化

Parnas(1972)核心准则: 模块分解标准应该是"需要隐藏的设计决策"，不是处理流程。列举"可能变更的设计决策"，每个决策分配给一个模块负责隐藏。

信号 — 模块边界画错了:
- 模块名带"-er"/"-or"后缀(Validator, Sender) → 过程式思维
- 一个功能变更需修改多个模块 → 信息未被隐藏
- 模块间共享全局数据结构 → 耦合过紧

模块边界怎么画:
1. 列举系统中"可能变更的设计决策"
2. 每个决策分配给一个模块隐藏
3. 检验: 模块名是领域名词而非动词
4. 检验: 单个需求变更只影响一个模块

层次分明原则:
- 避免循环依赖 — 分不清谁上谁下是最严重的设计问题
- 避免越层调用 — A→B→C中A不应直接调用C(基础设施模块除外)

C语言模块化标准模式:

```c
// module.h — 只声明，不定义
struct Module;                              // 不透明类型
struct Module* Module_create(void);
void Module_destroy(struct Module* self);
void Module_doWork(struct Module* self);

// module.c — 定义与实现
struct Module {                             // 结构体定义仅在实现文件中
    int internal_state;
    void* private_data;
};
```

关键规则:
- 不在module.h中`#include`依赖的实现头文件，用前向声明
- 所有私有函数和变量必须用`static`
- 前缀命名空间(`Module_xxx`)替代语言级namespace
- 用`nm module.o | grep " [A-Z] "`检查: 所有导出符号都应有模块前缀

C++ Pimpl — 编译防火墙:

```cpp
// widget.h — 公共头文件
class Widget {
public:
    Widget();
    ~Widget();
    void doSomething();
private:
    struct Impl;
    std::unique_ptr<Impl> pimpl_;           // 隐藏全部实现细节
};

// widget.cpp
struct Widget::Impl {
    // 平台特定代码、私有成员、依赖头文件全在这里
};
```

优势: 信息隐藏、减少编译依赖、二进制兼容(对象始终sizeof(pointer))。
代价: 额外堆分配、间接访问开销。

### 3. 契约设计

契约 = 前置条件(调用者义务) + 后置条件(实现者义务) + 不变量(双方义务)。

关键区分:
- Error(bug): 设计缺陷，不可"处理"，硬失败(hard fail)
- Exceptional condition: 合法但罕见的运行时情况，需要恢复策略

参数分两类:
- 内部输入(模块间调用): 完全可信，用契约式(断言检测bug)
- 外部输入(用户/网络/文件): 完全不可信，用防御式(错误码/异常)

```cpp
// 内部API — 契约式: 违反即bug，硬失败
void ProcessBuffer(Buffer* buf, size_t offset) {
    REQUIRE(buf != nullptr);
    REQUIRE(offset < buf->size());
    // 专心做正确的事
}

// 外部API — 防御式: 不信任输入
Status ParseUserInput(const char* input, Config* out) {
    if (input == nullptr) return Status::InvalidArg;
    if (strlen(input) > MAX_INPUT_LEN) return Status::TooLong;
    // 验证通过后调用内部API
}
```

Assert框架(嵌入式/系统编程适用):

```c
#ifdef NASSERT                              // 与NDEBUG解耦，独立控制
#define ASSERT(ignore_)  ((void)0)
#define ALLEGE(test_)    ((void)(test_))    // 禁用时仍求值(有副作用的表达式)
#else
void onAssert__(char const *file, unsigned line);
#define DEFINE_THIS_FILE static char const THIS_FILE__[] = __FILE__
#define ASSERT(test_) \
    ((test_) ? (void)0 : onAssert__(THIS_FILE__, __LINE__))
#define ALLEGE(test_) ASSERT(test_)
#endif

#define REQUIRE(test_)   ASSERT(test_)      // 前置条件
#define ENSURE(test_)    ASSERT(test_)      // 后置条件
#define INVARIANT(test_) ASSERT(test_)      // 不变量
```

实践规则:
- 发布代码保留assertion — "练习穿救生衣，上场却不穿"是荒谬的
- `DEFINE_THIS_FILE`避免`__FILE__`字符串在每个assert处重复，节省ROM
- 强前置条件消除大量defensive检查代码

继承中的契约(LSP的机械化检验):
- 子类前置条件可以更弱(接受更多输入)
- 子类后置条件可以更强(保证更多输出)
- 违反此规则 = 违反LSP

当前C++实践 — GSL的Expects/Ensures:

```cpp
#include <gsl/gsl>
double sqrt(double x) {
    Expects(x >= 0);                        // 前置条件
    auto result = /* ... */;
    Ensures(result >= 0);                   // 后置条件(也防止优化器删除关键操作)
    return result;
}
```

---

## 二、编码前审视: 选择正确的结构

### 模式选择速查

| 我需要... | 模式 | 一句话 |
|-----------|------|--------|
| 行为随状态变化 | State Machine | 显式状态+事件驱动替代标志位 |
| 变化时通知他人 | Observer | 事件源→回调列表→遍历通知 |
| 算法可替换 | Strategy / lambda | 行为参数化 |
| 流程固定步骤可变 | Template Method | 基类骨架+子类填充 |
| 创建与使用解耦 | Factory | 工厂函数返回unique_ptr |
| 参数过多 | Builder | 链式设置→Build() |
| 条件链可扩展 | Chain of Responsibility | 对象链替代if-else级联 |
| 统一处理个体与组合 | Composite | 树形结构递归 |
| 异构集合多种操作 | Visitor / variant+visit | 双重派发 |
| 操作可撤销/排队 | Command | 操作封装为对象 |
| 简化复杂子系统 | Facade | 高层接口隐藏复杂性 |
| 接口不兼容 | Adapter | 转换已有接口 |
| 控制对象访问 | Proxy | 延迟初始化/权限/日志 |
| 3+分支处理相似逻辑 | 表驱动 | 数据替代条件逻辑 |
| 配对的获取/释放 | RAII | 构造获取，析构释放 |

行为型模式与lambda的关系 — 如果核心只是"把行为参数化"，用lambda。只有需要额外结构(undo、组合、生命周期管理、类型安全分发)时才用完整模式类:

| 模式 | 可用lambda替代? |
|------|----------------|
| Strategy | 是: `std::function` / lambda |
| Command(简单) | 是: `std::function<void()>` |
| Observer(简单) | 是: callback list |
| Template Method | 部分: 传入step函数 |
| Visitor | 部分: `std::variant` + `std::visit` |

### 状态机

本质: 用显式状态+事件驱动替代散布在代码各处的条件逻辑。

信号:
- 多个bool标志位相互配合(`isRunning && !isPaused && isReady`)
- 函数行为取决于"上次做了什么"
- 注释里出现"在XX状态下才能YY"

三个实现层级:

层级一 — 状态转移表(状态 <5，规则固定):

```cpp
enum State { IDLE, STARTED, STOPPED, STATE_COUNT };
enum Event { EVT_START, EVT_STOP, EVENT_COUNT };

struct Transition {
    State next;
    void (*action)(void* ctx);
};

static const Transition table[STATE_COUNT][EVENT_COUNT] = {
    /*             EVT_START              EVT_STOP            */
    /* IDLE    */ {{STARTED, on_start},  {IDLE,    nullptr}},
    /* STARTED */ {{STARTED, nullptr},   {STOPPED, on_stop}},
    /* STOPPED */ {{STARTED, on_start},  {IDLE,    on_reset}},
};

void dispatch(State& state, Event evt, void* ctx) {
    auto& t = table[state][evt];
    if (t.action) t.action(ctx);
    state = t.next;
}
```

层级二 — 函数指针分发(状态5-15，需要封装状态行为):

```c
typedef void (*StateHandler)(Fsm* me, Event const* e);

typedef struct Fsm {
    StateHandler state;                     // 当前状态 = 函数指针
} Fsm;

void fsm_dispatch(Fsm* me, Event const* e) {
    me->state(me, e);                       // 直接分发
}

void fsm_tran(Fsm* me, StateHandler target) {
    me->state = target;                     // 状态转移 = 换指针
}

void state_idle(Fsm* me, Event const* e) {
    switch (e->sig) {
        case START_SIG:
            do_start_action(me);
            fsm_tran(me, &state_running);
            break;
    }
}
```

层级三 — 层次状态机/HSM(状态 >15，有层次共性):

子状态未处理的事件上升到父状态，类似继承 — "programming by difference"。

```c
// HSM状态函数返回父状态(未处理)或NULL(已处理)
StateHandler state_vehiclesGreen(Fsm* me, Event const* e) {
    switch (e->sig) {
        case ENTRY_SIG: start_green_timer(me); return NULL;
        case EXIT_SIG:  stop_green_timer(me);  return NULL;
        case TIMEOUT_SIG:
            fsm_tran(me, &state_vehiclesYellow);
            return NULL;
        default:
            return &state_vehiclesEnabled;  // 未处理 -> 上升到父状态
    }
}
```

HSM中LSP的应用: 子状态不能违反父状态的语义契约。

陷阱:
- Guard condition滥用: 一个转移需要 >3个guard → 考虑增加状态替代
- 状态机 ≠ 流程图: 状态机是事件驱动(被动等待)，流程图是转换驱动(自动进入下一步)

### 工厂

信号: 代码中`new ConcreteClass`散布在业务逻辑中，switch/if-else根据类型字符串创建对象。

手动注册工厂(简洁实用):

```cpp
using CreateFn = std::function<std::unique_ptr<ITransport>()>;

class TransportFactory {
    std::unordered_map<std::string, CreateFn> creators_;
public:
    void Register(const std::string& name, CreateFn fn) {
        creators_[name] = std::move(fn);
    }
    std::unique_ptr<ITransport> Create(const std::string& name) {
        auto it = creators_.find(name);
        return it != creators_.end() ? it->second() : nullptr;
    }
};
```

自动注册工厂(零手动注册 — Unforgettable Factory):

```cpp
template <class Base, class... Args>
class Factory {
public:
    template <class T>
    struct Registrar : Base {
        static bool registered;
        static bool registerT() {
            Factory::data().emplace(typeid(T).name(),
                [](Args... args) -> std::unique_ptr<Base> {
                    return std::make_unique<T>(std::forward<Args>(args)...);
                });
            return true;
        }
        Registrar() { (void)registered; }   // 强制实例化静态变量
    };

    static std::unique_ptr<Base> Make(const std::string& name, Args&&... args) {
        auto& m = data();
        auto it = m.find(name);
        return it == m.end() ? nullptr : it->second(std::forward<Args>(args)...);
    }
private:
    static auto& data() {
        static std::unordered_map<std::string,
            std::function<std::unique_ptr<Base>(Args...)>> s;
        return s;
    }
};

template <class Base, class... Args>
template <class T>
bool Factory<Base, Args...>::Registrar<T>::registered =
    Factory<Base, Args...>::Registrar<T>::registerT();

// 使用: 继承Registrar即自动注册
class TcpTransport : public Factory<ITransport>::Registrar<TcpTransport> {
    void Send(const void* data, size_t len) override { /* ... */ }
};
```

Singleton(全局唯一资源):

C++11唯一推荐写法 — Meyer's Singleton:

```cpp
class DeviceManager {
public:
    static DeviceManager& GetInstance() {
        static DeviceManager instance;      // C++11保证线程安全
        return instance;
    }
    DeviceManager(const DeviceManager&) = delete;
    DeviceManager& operator=(const DeviceManager&) = delete;
private:
    DeviceManager() = default;
};
```

CRTP泛化版(系统中单例多时):

```cpp
template <typename Derived>
class Singleton {
public:
    static Derived& GetInstance() {
        static Derived instance;
        return instance;
    }
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
protected:
    Singleton() = default;
    ~Singleton() = default;
};

class ConfigManager : public Singleton<ConfigManager> {
    friend Singleton<ConfigManager>;
private:
    ConfigManager() = default;
};
```

单例陷阱: (1)析构顺序不确定，多个单例互相依赖会出问题 (2)隐式全局状态使测试困难。如果仅为"方便访问"，考虑依赖注入。

### 观察者

信号: 一个对象改变后需手动调用多个其他对象的更新方法，注释写着"别忘了也更新XX"。

安全实现(解决三个致命缺陷):

```cpp
class Observable {
    std::set<Observer*> observers_;
public:
    void attach(Observer* o)  { observers_.insert(o); }
    void detach(Observer* o)  { observers_.erase(o); }
protected:
    void notify() {
        auto snapshot = observers_;             // 1. 快照防止遍历中修改
        for (auto* o : snapshot) {
            if (observers_.count(o))            // 2. 双重检查是否已移除
                o->update(*this);
        }
    }
};

class SafeObserver {
    std::set<Observable*> attached_to_;
public:
    virtual ~SafeObserver() {
        for (auto* obs : attached_to_)          // 3. 析构时自动断开
            obs->detach(this);
    }
};
```

嵌入式场景(ETL零堆分配):

```cpp
class SensorDriver : public etl::observable<SensorObserver, 4> {
    void on_data_ready(int val) {
        notify_observers(val);              // 编译期固定最大观察者数
    }
};
```

性能陷阱: 模板观察者导致库从5MB膨胀到16MB。解法: 内部用`void*`列表 + 类型安全外部接口。

推模型(notify推数据)耦合高但减少延迟，拉模型(observer回调查询)灵活。系统编程中常用推模型。

### 结构型模式速查

Facade — 为复杂子系统提供简化高层接口:

```cpp
class CommEngine {
public:
    Status Initialize(const Config& cfg) {
        auto ret = driver_.Open(cfg.device_id);
        if (ret != OK) return ret;
        ret = allocator_.AllocateBuffers(cfg.buffer_count, cfg.buffer_size);
        if (ret != OK) return ret;
        return stream_.Setup(driver_, allocator_);
    }
private:
    DeviceDriver driver_;
    MemoryAllocator allocator_;
    StreamManager stream_;
};
```

Adapter — 转换不兼容接口(系统编程中极常见: 封装C库为C++接口):

```cpp
class SocketAdapter : public ICommChannel {
    LegacySocket& socket_;
public:
    int Send(const void* buf, size_t len) override {
        return socket_.TransmitData(
            static_cast<const char*>(buf), static_cast<int>(len));
    }
};
```

Composite — 统一处理个体与组合(树形/层次结构):

```cpp
class ITopologyNode {
public:
    virtual void Execute(const Task& task) = 0;
    virtual ~ITopologyNode() = default;
};
class DeviceNode : public ITopologyNode { /* 叶节点 */ };
class DeviceGroup : public ITopologyNode {
    std::vector<std::unique_ptr<ITopologyNode>> children_;
public:
    void Execute(const Task& task) override {
        for (auto& child : children_)
            child->Execute(task);           // 递归: 不区分叶和组合
    }
};
```

Proxy — 控制对象访问(延迟初始化/权限/日志):

```cpp
class ComputeProxy : public ICompute {
    std::unique_ptr<RealCompute> real_;
public:
    Status Execute(const Tensor& in, Tensor& out) override {
        if (!real_) real_ = std::make_unique<RealCompute>();  // lazy init
        auto start = std::chrono::steady_clock::now();
        auto ret = real_->Execute(in, out);
        auto elapsed = std::chrono::steady_clock::now() - start;
        LOG("Execute took %lld us", /*...*/);                 // 计时
        return ret;
    }
};
```

Decorator — 动态附加职责(比子类继承更灵活):

```cpp
// 动态装饰器: 层层包装
auto stream = std::make_shared<EncryptedStream>(
    std::make_shared<BufferedStream>(
        std::make_shared<FileStream>("log.bin")));

// 静态装饰器(CRTP Mixin, 零开销):
template <typename T>
struct ColoredShape : public T {
    std::string color;
    std::string GetName() const { return T::GetName() + " [" + color + "]"; }
};
using RedCircle = ColoredShape<Circle>;
```

Bridge — 防止笛卡尔积类爆炸(两个独立变化维度):
M种消息 x N种传输只需M+N个类。Pimpl是Bridge的退化形式。

低实用性模式(一句话):
- Prototype: `virtual clone()` — 只有基类指针需要派生类副本时
- Flyweight: 共享不变状态节省内存 — `std::string` SSO、符号表
- Memento: 状态快照/回滚 — 系统编程中通常用序列化/checkpoint替代

### 表驱动法

本质: 数据压倒一切。将代码复杂度转移到数据中。

信号: 3+分支处理相似逻辑、状态转移、消息分发、命令路由。

三种查表方式:

```c
// 直接查找 — O(1)
char n2c = "0123456789ABCDEF"[num];

// 索引查找 — 消息分发
typedef struct {
    uint32_t    msgCode;
    MsgHandler  handler;
} MsgDispatchEntry;

static const MsgDispatchEntry gDispatchTable[] = {
    { MSG_INIT,     HandleInit },
    { MSG_DATA,     HandleData },
    { MSG_SHUTDOWN, HandleShutdown },
};

// 分段查找 — 区间映射
typedef struct { double limit; const char* grade; } GradeMap;
static const GradeMap gGradeMap[] = {
    {50.0, "Fail"}, {60.0, "Pass"}, {70.0, "Credit"},
    {80.0, "Distinction"}, {100.0, "High Distinction"},
};
```

数据驱动的三个层次: 函数级(查找表替代分支) → 程序级(表驱动状态机) → 系统级(DSL/配置文件驱动)。

注意: C中函数指针不携带类型信息 — 参数和返回值类型不受编译器检查。C++中用`std::function`或模板获得类型安全。

---

## 三、编码中遵循: 接口与实现

### 接口设计原则

命令查询分离(CQS):
- Query: 返回结果，不改状态(标记const)
- Command: 改变状态，不返回值(返回void)
- "Asking a question should not change the answer."

```cpp
// BAD: 违反CQS — pop()混合了command和query
template<typename T>
T Stack<T>::pop() {
    T val = top_;                           // 拷贝构造可能抛异常
    --size_;                                // 元素已移除，val却丢失
    return val;                             // 违反强异常安全保证
}

// GOOD: 分离
const T& top() const;                      // query
void pop();                                // command
```

合理违反CQS的场景: 原子操作(compare_exchange)、Fluent Interface、迭代器推进。

API六个质量特征(综合Qt + Martin Reddy):
1. 最小化: "有疑问时，留在外面" — 添加比删除容易
2. 完整: 预期功能必须存在
3. 语义清晰: 最小惊讶原则
4. 直觉化: 中等经验用户无需文档
5. 易记忆: 一致精确的命名，永远不缩写
6. 导向可读代码: 读的次数远多于写

```cpp
// BAD: 密码式参数
QSlider* s = new QSlider(12, 18, 3, 13, Qt::Vertical, 0, "volume");

// GOOD: 意图清晰
QSlider* s = new QSlider(Qt::Vertical);
s->setRange(12, 18);
s->setPageStep(3);
s->setValue(13);
```

```cpp
// BAD: bool参数含义不明
widget->repaint(false);                     // false是什么意思?

// GOOD: enum替代bool
str.replace("USER", user, Qt::CaseInsensitive);
```

降低耦合:
- 前向声明优先于#include
- 非成员非友元函数优先于成员函数 — 提高封装性
- 永远不返回非const指针/引用指向private数据
- virtual函数谨慎添加 — 无法在不破坏二进制兼容性的情况下删除

### 依赖注入

默认用构造函数注入(适配RAII):

```cpp
class MessageProcessor {
public:
    explicit MessageProcessor(
        std::unique_ptr<ITransport> transport,
        std::unique_ptr<ISerializer> serializer)
        : transport_(std::move(transport))
        , serializer_(std::move(serializer)) {}  // 对象始终合法
};
```

性能敏感路径用模板参数(零开销):

```cpp
template <typename Transport, typename Serializer>
class MessageProcessor {
    void Process(const Message& msg) {
        auto data = Serializer::Serialize(msg);
        Transport::Send(data);              // 编译时绑定，可内联
    }
};
```

DI vs Service Locator:
- 选DI: 编写供他人使用的库(不知道用户用什么Locator)
- 选Service Locator: 应用内部代码，部署环境已知

### OOP in C

Linux内核、嵌入式系统、CANN底层C代码都在用C实现面向对象。

封装 — 不透明指针:

```c
// device.h
typedef struct Device Device;              // 用户看不到内部
Device* device_create(int id);
void device_destroy(Device* dev);
int device_send(Device* dev, const void* data, size_t len);

// device.c
struct Device { int id; int fd; void* buf; };
```

接口 — 函数指针结构体(vtable):

```c
struct transport_ops {
    int (*open)(struct transport* t, const char* addr);
    int (*send)(struct transport* t, const void* data, size_t len);
    void (*close)(struct transport* t);
};
```

继承 — 结构体内嵌:

```c
struct transport {
    const struct transport_ops* ops;        // vtable指针
    int ref_count;
};

struct tcp_transport {
    struct transport base;                  // "继承": 第一个成员
    int sockfd;
};

// 向上转型: 安全
struct transport* t = (struct transport*)tcp;

// 向下转型: container_of宏
#define container_of(ptr, type, member) \
    ((type*)((char*)(ptr) - offsetof(type, member)))
struct tcp_transport* tcp = container_of(t, struct tcp_transport, base);
```

多态 — 虚函数分发:

```c
// NVI wrapper
static inline int transport_send(struct transport* t,
                                  const void* data, size_t len) {
    return t->ops->send(t, data, len);
}

// 具体实现
static int tcp_send(struct transport* t, const void* data, size_t len) {
    struct tcp_transport* tcp = container_of(t, struct tcp_transport, base);
    return write(tcp->sockfd, data, len);
}

// "构造函数"中绑定vtable
static const struct transport_ops tcp_ops = {
    .open  = tcp_open,
    .send  = tcp_send,
    .close = tcp_close,
};

struct tcp_transport* tcp_transport_create(void) {
    struct tcp_transport* tcp = calloc(1, sizeof(*tcp));
    tcp->base.ops = &tcp_ops;
    return tcp;
}
```

Linux内核OO惯例:
- vtable命名: `<object>_operations`或`<object>_ops`
- 可空函数指针: 允许渐进式开发，调用者检查NULL提供默认行为
- Mixin: 额外ops结构体为对象附加可选能力

### C迭代器接口

三种模式，按场景选择:

```c
// 模式一: Index Access — 简单数组，需要随机访问
int    count(Collection* c);
Item*  get(Collection* c, int index);

// 模式二: Cursor (first/done/next) — 通用容器，对外API(默认选择)
Item* first(Collection* c);
bool  done(Collection* c, Item* cursor);
Item* next(Collection* c, Item* cursor);

// 模式三: Callback — 需要遍历原子性(持锁遍历)
typedef void (*ItemCallback)(Item* item, void* user_data);
void foreach(Collection* c, ItemCallback cb, void* user_data);
```

### RAII与资源管理

RAII(C++最核心的模式):
- 信号: 任何需要配对的获取/释放操作
- 做法: 构造函数获取，析构函数释放
- 场景: stream/event/buffer/lock的管理

Scope Guard — 不值得单独建类时:

```cpp
auto guard = std::unique_ptr<void, std::function<void(void*)>>(
    resource, [](void* r) { release(r); });
// 或C++17 std::optional + 析构lambda
```

Placement new与析构责任:
- 在已分配内存上构造对象(内存池/allocator核心机制)
- 必须手动调析构函数`obj->~T()`，不能delete
- 对齐责任完全由使用者承担

### 现代C++特性优先级

C++14:
- auto返回类型推导
- 泛型lambda
- `[[deprecated]]`

C++17:
- structured bindings: `auto [key, value] = pair;`
- if constexpr: 编译期分支消除
- std::optional: 替代裸指针表示"可能为空"
- std::string_view: 零拷贝字符串引用
- `[[nodiscard]]`: 防止忽略返回值
- 折叠表达式: 替代递归模板

C++20:
- concepts: 约束模板参数
- std::span: 替代(ptr, size)参数对
- designated initializers: `.field = value`
- `[[likely]]`/`[[unlikely]]`: 分支预测提示

C++11(后续各节详述):
- noexcept: 移动构造/赋值必须标记(无noexcept时vector扩容慢10倍，详见值语义节)
- enum class: 强类型枚举，不隐式转int，不污染命名空间
- 初始化三板斧(#88): `=`赋值/字面量, `()`构造逻辑, `{}`仅在前两者不行时
- static局部变量初始化线程安全(Meyer's Singleton的基础)

### 多态实现方式选择

信号: virtual/CRTP/Type Erasure/Concepts/variant之间需要决策。

```
需要运行时多态(异构容器/插件)?
├─ 是 → 能修改源码?
│   ├─ 否 → Type Erasure (std::function / Proxy)
│   └─ 是 → 类型集合封闭?
│       ├─ 是 → std::variant + std::visit
│       └─ 否 → virtual继承
└─ 否 → C++20可用?
    ├─ 是 → Concepts
    └─ 否 → CRTP
```

| 维度 | virtual | CRTP | Type Erasure | Concepts | variant |
|------|---------|------|-------------|----------|---------|
| 绑定 | 运行时 | 编译期 | 运行时 | 编译期 | 运行时 |
| 异构容器 | 可 | 不可 | 可 | 不可 | 可 |
| 侵入性 | 需继承 | 需继承 | 不需要 | 不需要 | 不需要 |
| 运行开销 | vptr+间接调用 | 零 | 间接调用 | 零 | 无间接调用 |
| 扩展方向 | 新类型容易 | 编译期固定 | 新类型容易 | 编译期固定 | 新操作容易 |

virtual要点:
- 析构函数二选一: public virtual(允许基类指针delete) 或 protected non-virtual(禁止)
- final触发devirtualization: `class Leaf final : public Base {}`
- 协变返回类型: override可返回更具体的指针/引用(仅限裸指针，不适用智能指针)
- 构造/析构函数中调虚函数不发生动态绑定

CRTP(参见DIP节的Type Erasure对比):
- `class D : Base<D>` — 基类通过`static_cast<Derived*>(this)`调用派生类
- 6大应用: 静态多态、运算符扩展(只实现`<`自动获得全部比较)、对象计数、链式多态、Facade、Template Method
- C++20后Concepts可替代大部分场景
- `Base<D1>`和`Base<D2>`是不同类型，不能放入同一容器

Type Erasure(参见DIP节):
- `std::function`是最常用的类型擦除; `shared_ptr`的deleter也是类型擦除
- unique_ptr的Deleter是模板参数(影响类型); shared_ptr的不是(类型擦除)

### 值语义与移动语义

信号: 避免不必要的深拷贝，理解std::move行为。

std::move的本质: 不移动任何东西，只是`static_cast<T&&>`。真正的资源转移发生在移动构造/赋值被调用时。

五大误区:

| 误区 | 现象 | 检测 |
|------|------|------|
| return中move | 阻止NRVO，多一次move | PVS V828 |
| 对const move | 静默退化为拷贝 | PVS V833 |
| move后用源对象 | valid but unspecified状态 | 审查 |
| 基本类型move | 值不变(退化为拷贝) | 无 |
| 返回const值类型 | 阻止move赋值 | PVS V839 |

```cpp
// 错: return中move阻止NRVO
std::string f() { std::string s = "x"; return std::move(s); }
// 对: 让编译器做NRVO
std::string f() { std::string s = "x"; return s; }

// 错: const对象move静默退化为拷贝(编译器不报错!)
const std::vector<int> data = getData();
consume(std::move(data));                // 实际执行拷贝
```

noexcept的关键性: 移动构造/赋值必须标noexcept。vector扩容时若move可能抛异常则回退为逐个拷贝。

| 操作 | 耗时 |
|------|------|
| 有noexcept的扩容 | 1.63 ms |
| 无noexcept的扩容 | 16.42 ms (慢10倍) |

```cpp
Resource(Resource&& other) noexcept
    : data(std::exchange(other.data, nullptr)),
      size(std::exchange(other.size, 0)) {}
```

Rule of Five: 自定义了析构函数 → 必须同时定义拷贝构造、拷贝赋值、移动构造、移动赋值。
Rule of Zero: 不直接管理资源(用智能指针RAII包装) → 不自定义任何特殊成员函数。

值类别: lvalue(有地址可访问) / xvalue(有地址但标记过期，std::move产生) / prvalue(无地址，字面量)。注意右值引用变量本身是左值: `int&& r = 42;`中r有名字、可取地址。

陷阱:
- SSO: 小字符串移动仍需拷贝字节
- 虚函数有noexcept则覆盖函数必须也写noexcept

### 对象模型与内存布局

信号: 优化结构体大小、理解sizeof结果、跨进程通信的数据对齐。

对齐规则: 自对齐(char=1, short=2, int=4, ptr=8)。结构体对齐 = 最大成员对齐值。

```cpp
struct Bad  { char c; long* p; short x; };    // 24字节
struct Good { long* p; short x; char c; };    // 16字节
// 按对齐值递减排列消除内部padding
```

sizeof速查:

| 场景 | sizeof |
|------|--------|
| 空叶子类 | 1 (每个对象需唯一地址) |
| 空基类(EBO) | 0 (不占派生类空间) |
| 有虚函数(不管几个) | +1个vptr(64位=8字节) |
| 静态成员 | 不算(跟类走不跟对象走) |
| 非静态成员函数 | 不变(不占对象空间) |

陷阱:
- `memcmp`比较结构体不安全(padding值未定义)
- `#pragma pack`生成更慢代码，仅用于匹配硬件/协议布局
- false sharing: 多线程读写数据分到不同cache line(x86 = 64字节)

### 泛型编程

信号: 模板编译报错难理解、需要编译期类型选择。

两阶段名字查找:

```cpp
template<typename T>
class Derived : public Base<T> {
    void Func() {
        Method();              // 错: 非依赖名，第一阶段找不到
        this->Method();        // 对: 变成依赖名，延迟到第二阶段
        // Base<T>::Method();  // 编译通过但变成静态绑定! 纯虚函数则崩溃
    }
};
```

SFINAE: 模板形参即使不在函数体中使用也承担编译期语义(enable_if、CRTP占位、ABI兼容预留)。C++20用concepts更直观:

```cpp
// SFINAE
template <class T>
auto print(T v) -> std::enable_if_t<std::is_integral_v<T>> { ... }

// C++20 concepts
template <std::integral T> void print(T v) { ... }
```

auto推导规则:
- 非指针/非引用声明: 抛弃引用 + 抛弃cv
- 指针/引用声明: 抛弃引用 + 保留cv

```cpp
const int& cr = 42;
auto a = cr;           // int (抛cv+引用)
auto& b = cr;          // const int& (保留cv)
auto* d = &cr;         // const int* (保留cv)
```

ADL(Argument-Dependent Lookup): 调用`func(a)`时编译器也在a类型所属namespace查找。对STL算法务必全限定: `std::count()`不要`count()`。

### C底层技术

信号: 系统编程中的C宏惯用法、位操作。

do-while(0)宏:

```c
#define RETURN_IF(C, R) do { if (!!(C)) { return R; } } while ((void)0, 0)
// do-while(0): 多语句宏在任何控制流中表现为单条语句
// !!: 非零值规范化为0/1
// (void)0,0: 抑制MSVC C4127警告
```

container_of(参见OOP in C节详细用法): 从成员地址反推宿主地址，侵入式数据结构核心。

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
// 必须转char*，否则指针算术按所指类型大小计算
```

位操作核心: `n & 1`判奇偶, `n && !(n & (n-1))`判2的幂, `byte |= (1<<n)`设位, 全部XOR找唯一数。

C/C++混编: `extern "C" {}`禁用name mangling。不能在其内部`#include`其他头文件(会把C++符号强制为C链接)。

陷阱:
- 宏参数有副作用时双重求值: `MIN(x++, y)` → x++执行两次
- 位操作优先级低于比较: `n & (n-1) == 0`解析为`n & ((n-1) == 0)`, 必须加括号
- likely/unlikely在现代CPU上效果有限，仅极端热路径有可测量效果

### 翻译单元与物理设计

信号: 编译依赖过重、增量构建慢、链接时undefined reference。

链接性:
- 翻译单元 = 经过预处理的源文件
- static在任何位置都有内部链接性
- const: C中外部链接性，C++中内部链接性 → "用constexpr替代宏"仅限C++

extern四种含义:

| 语境 | 含义 |
|------|------|
| 非const全局变量 | 声明另一翻译单元中的定义 |
| const全局变量(C++) | 使const具有外部链接性: `extern const int i = 10;` |
| `extern "C"` | 禁用name mangling，C调用约定 |
| 模板声明 | 复用其他位置的实例化(避免重复) |

static的多重语义:
- 全局/namespace: 内部链接性
- 块作用域: 静态存储期，首次经过时初始化
- C++11起: 静态局部变量初始化线程安全(只发生一次)

头文件规则:
- 放声明不放定义; 可放: inline函数、类、const对象(仅C++)、static
- 禁止: `using namespace`(传染性)、`#pragma pack`(传染性)
- Include Guard保证幂等; 对应.cpp第一行include自己的.h(验证自给自足)

前向声明决策:
- 指针/引用/函数参数/返回类型 → 前向声明
- 继承/值成员/sizeof → #include
- 什么也不依赖/friend声明 → 什么也不做

```cpp
#ifndef MODULE_H_
#define MODULE_H_

class Foo;                             // 前向声明
#include <vector>                      // 值成员需要
#include "parent.h"                    // 继承需要

class MyClass : public Parent {
    std::vector<int> data;             // 值成员: #include
    Foo* foo;                          // 指针: 前向声明
    void Func(Bar& bar);              // 引用: 前向声明
};

#endif
```

命名空间卫生:
- 匿名namespace替代static限制符号可见性(#186)
- 禁止`using namespace`(#153): 名字注入最近公共祖先，产生二次方碰撞风险
- using声明用全限定名(#119): `using ::lib::Type;`
- 顶级namespace保持扁平(#130): 嵌套越深，名字查找意外匹配层数越多
- 头文件中inline constexpr(#140): `inline constexpr int kVal = 42;` 确保全程序单一实例

编译优化: ccache(热缓存最快) / Pimpl(改实现不重编依赖方，参见信息隐藏节) / PCH(cmake 3.16+的`target_precompile_headers`) / iwyu(头文件依赖分析)

### 错误处理与参数传递

信号: 函数可能失败时的返回值设计、接口参数选择。

Status/StatusOr(Abseil #76/#181):

```cpp
absl::Status DoWork();
DoWork();                                // 编译错误: 不允许忽略
DoWork().IgnoreError();                  // 显式忽略，意图清晰

if (auto foo = TryCreate(); foo.ok()) {
    foo->DoBar();                        // StatusOr用operator->访问
}
// 避免.value(): 行为依赖编译模式(异常 vs LOG(FATAL))
```

避免哨兵值(#171): 用`std::optional`替代magic number。哨兵值本身可能是合法域值，未检查时静默传播损坏结果。

参数传递(#234):

| 场景 | 方式 |
|------|------|
| 数值/枚举/sizeof<=16 trivial | by value |
| 字符串只读 | string_view |
| vector只读 | Span<const T> |
| 转移所有权 | unique_ptr by value |
| 大对象只读 | const T& |
| 可选参数 | const T* (nullable) |
| 需自己副本的容器 | by value + std::move |

参数过多时用Option Struct(#173):

```cpp
struct Options { int precision = 8; bool scientific = false; };
void Print(double val, const Options& opts = {});
Print(5.0, {.precision = 2, .scientific = true});  // C++20
```

optional vs unique_ptr(#123): 裸对象(总是可用) → optional(可延迟构造，cache友好) → unique_ptr(需转移所有权或多态)。避免`const std::optional<Foo>&`作参数(#163): 传入裸Foo时会拷贝到临时optional，用`const Foo*`代替。

Flags是全局变量(#45): 库代码禁止Flags，通过函数参数显式传入配置。

### STL容器实战

信号: 排序谓词、容器操作的常见陷阱。

排序谓词统一记忆:
- C++: `cmp(a,b)` 返回true不交换 → `a<b`为true时a在前 → 升序
- C: `cmp(a,b)` 返回正数不交换 → `a>b`为正时b在前(a,b位置与C++相反) → 也是升序
- 本质: 返回"已经有序"则不动

```c
// C qsort安全比较(避免溢出)
int cmp(const void *a, const void *b) {
    int x = *(const int*)a, y = *(const int*)b;
    return (x > y) - (x < y);             // 安全
    // 不要用 return x - y;               // INT_MIN-1溢出
}
```

lower_bound vs upper_bound: lower返回`>=x`的第一个位置，upper返回`>x`的第一个位置。`[lower, upper)` = 所有等于x的范围。

emplace_back vs push_back(#112): 默认push_back。emplace_back绕过explicit检查:

```cpp
std::vector<std::regex> v;
v.push_back(nullptr);             // 编译错误(好!)
v.emplace_back(nullptr);          // 编译通过，运行崩溃
```

容器陷阱:
- `v.size() - 1`空容器回绕(#227): 改写为`i + 1 < v.size()`
- 不要用`at()`(#224): 异常不是好的越界错误处理
- flat_hash_map(#136): 新代码优先; `map["a"] = map["b"]`可能rehash后访问失效内存
- reader lock(#197): 短临界区用独占锁; reader lock的簿记开销通常大于争用节省

C++回调兼容C函数指针: 生产首选void* userdata模式(支持多实例、无全局状态); 无捕获lambda可隐式转函数指针，有捕获则不行。

---

## 四、编码后回看: 识别改进机会

### 代码smell

函数级:
- 过长(>40行): 拆分为子函数，每个做且只做一件事
- 过深嵌套(>3层): early return / guard clause扁平化
- 参数过多(>4个): 结构体/Config对象打包，或Builder
- 重复代码: 两处以上 → 提取公共函数；相似但有差异 → 模板或Strategy

命名级:
- 函数名不自解释: 好`CalculateGcdGroupSize(counts)` / 差`calc(v)`
- 魔数: 所有字面量用`constexpr`命名常量
- bool参数: `CreateComm(true, false)` → `CreateComm(Mode::kAsync, Verify::kDisabled)`

设计级:
- 上帝类(>500行或>10个public方法): 拆分职责
- Feature Envy(频繁访问另一个类的数据): 将方法移到数据所在的类
- 数据泥团(同组参数反复出现): 提取为结构体

### 重构手法速查

| 手法 | 时机 |
|------|------|
| Extract Method | 最常用: 代码块可用动词+名词命名 |
| Rename | 最有效: 让名字说出意图 |
| Replace Conditional with Polymorphism | 分支>3且可能扩展 |
| Introduce Parameter Object | 参数>4 |
| Replace Magic Number | 所有字面量 |
| Extract Class | 类>500行 |
| Move Method | 方法放错了类 |

---

## 附录: 包设计原则

类级别之上，还有包级别的设计原则(直接映射到头文件组织、库划分、编译依赖管理):

内聚性(包里放什么):
- REP(发布复用等价): 复用粒度 = 发布粒度
- CCP(共同闭包): 因同一原因变化的类打包在一起(SRP的包级版本)
- CRP(共同复用): 经常一起使用的类打包在一起

耦合性(包间关系):
- ADP(无环依赖): 依赖图不能有环
- SDP(稳定依赖): 依赖方向朝稳定性增加的方向
- SAP(稳定抽象): 越稳定的包应越抽象
