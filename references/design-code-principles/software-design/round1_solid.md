# Round 1: SOLID原则深化 — C++系统编程实战指引

> 源材料: Notion软件工程知识库 + Uncle Bob博客 + vishalchovatiya C++ SOLID系列 + hackernoon/reflectoring等外部资源

---

## 1. 单一职责原则 (SRP — Single Responsibility Principle)

### 本质

一个模块应该只对一个actor(利益相关者)负责。"只有一个改变的理由"中的"理由"不是技术原因，是人 — 是不同的业务角色提出了不同的变更需求。

> Gather together the things that change for the same reasons. Separate those things that change for different reasons. — Robert C. Martin

### 违反时的代码信号

- 构造函数参数超过5~6个 — 承担了过多职责，或遗漏了Builder模式
- 一个类同时处理数据序列化和业务逻辑
- 一个函数既做计算又做I/O(如: 计算拓扑的同时写日志)
- 修改一个功能时，不相关的测试开始失败
- 类名中出现"Manager"、"Handler"、"Processor"、"Utils"等模糊词汇

### 遵循时的机制

1. 识别actor: 这段代码服务于哪些不同的利益相关者？
2. 按actor划分职责: 将因同一actor需求变化的代码聚合在一起
3. 分离变化轴: 将因不同原因变化的逻辑拆到不同的类/模块中
4. Facade聚合: 拆分后如果外部需要统一入口，用Facade组合各职责类

### C++系统编程典型场景

场景: 通信库中的连接管理类同时负责socket生命周期、序列化协议、重试策略

```cpp
// 违反SRP: 一个类做了三件事
class Connection {
public:
    void connect(const Endpoint& ep);
    void disconnect();
    Buffer serialize(const Message& msg);    // 序列化 — 不同的变化原因
    Message deserialize(const Buffer& buf);
    void sendWithRetry(const Message& msg);  // 重试策略 — 又一个变化原因
private:
    Socket sock_;
    RetryConfig retryConf_;
    SerializeFormat fmt_;
};

// 遵循SRP: 按变化原因拆分
class Transport {                            // actor: 网络基础设施团队
public:
    void connect(const Endpoint& ep);
    void disconnect();
    void send(const Buffer& buf);
    Buffer recv();
private:
    Socket sock_;
};

class Codec {                                // actor: 协议团队
public:
    Buffer encode(const Message& msg);
    Message decode(const Buffer& buf);
};

class RetryPolicy {                          // actor: 可靠性团队
public:
    template<typename F>
    auto execute(F&& fn) -> decltype(fn());  // 可独立变化重试策略
};

// Facade: 对外提供统一接口
class Connection {
public:
    Connection(Transport t, Codec c, RetryPolicy r);
    void send(const Message& msg) {
        auto buf = codec_.encode(msg);
        retry_.execute([&]{ transport_.send(buf); });
    }
private:
    Transport transport_;
    Codec codec_;
    RetryPolicy retry_;
};
```

### 常见误解和陷阱

- 过度拆分: SRP不是通往最小类的单行道。如果拆分后一个职责只有3行代码且不会独立变化，不值得独立成类。SRP提出的是聚合与分离之间的平衡点
- 过早应用: 不要在第一次编写时就预测所有可能的actor。当观察到一个类因不同原因反复变化时再拆分
- 与内聚性的关系: SRP是高内聚的自然结果，不是目标本身。先确保类有一个清晰的、与其名称匹配的更高层次目的
- Parnas的洞察: 模块划分不应基于流程图(执行步骤)，而应基于"可能变化的设计决策"。每个模块隐藏一个设计决策

---

## 2. 开放封闭原则 (OCP — Open-Closed Principle)

### 本质

你应该能够在不修改已有代码的情况下扩展系统的行为。新功能通过编写新代码来添加，而非修改旧代码。

> You should be able to extend the behavior of a system without having to modify that system. — Robert C. Martin

### 违反时的代码信号

- `if/else`或`switch`链根据类型做不同处理，且每次新增类型都要修改这个链
- 直接依赖具体类而非接口/抽象，导致新增实现时必须改调用方
- 修改一个模块的实现需要重新编译其调用方(头文件中暴露了实现细节)
- 霰弹枪手术(Shotgun Surgery): 添加一个新功能需要在多个文件中做修改

### 遵循时的机制

六种C++实现手段:

1. 动态多态(virtual + 继承): 最经典的方式，运行时绑定，对添加新类型开放
2. 静态多态(模板): 编译时绑定，零开销抽象
3. CRTP(Curiously Recurring Template Pattern): 编译期多态，零vtable开销
4. 策略模式(std::function / 函数对象): 灵活注入行为
5. 模板方法模式: 基类定义骨架，子类填充步骤
6. std::variant + std::visit: 对添加新操作开放(与virtual方向相反，适合类型固定但操作可变的场景)

### C++系统编程典型场景

场景: 集合通信库需要支持多种通信算法(ring, recursive halving, pairwise exchange)，且可能随时新增算法

```cpp
// 违反OCP: switch on algorithm type
void allReduce(void* buf, size_t count, AlgoType algo) {
    switch (algo) {
        case AlgoType::Ring:        ringAllReduce(buf, count); break;
        case AlgoType::RecursiveHD: recursiveHDAllReduce(buf, count); break;
        // 每次新增算法都要改这里
    }
}

// 方案A: 动态多态 — 运行时可切换算法
class AllReduceAlgo {
public:
    virtual ~AllReduceAlgo() = default;
    virtual void execute(void* buf, size_t count, Communicator& comm) = 0;
};

class RingAllReduce : public AllReduceAlgo {
public:
    void execute(void* buf, size_t count, Communicator& comm) override;
};

// 新增算法: 只写新代码，不改已有代码
class RecursiveHDAllReduce : public AllReduceAlgo {
public:
    void execute(void* buf, size_t count, Communicator& comm) override;
};

void allReduce(void* buf, size_t count, AllReduceAlgo& algo, Communicator& comm) {
    algo.execute(buf, count, comm);
}
```

```cpp
// 方案B: 静态多态(模板) — 编译时绑定，零开销
template<typename Algo>
void allReduce(void* buf, size_t count, Communicator& comm) {
    Algo algo;
    algo.execute(buf, count, comm);
}

// 使用:
allReduce<RingAllReduce>(buf, count, comm);
```

```cpp
// 方案C: std::function — 最灵活，可组合
using AllReduceFn = std::function<void(void* buf, size_t count, Communicator& comm)>;

void allReduce(void* buf, size_t count, Communicator& comm, AllReduceFn algo) {
    algo(buf, count, comm);
}

// 可以用lambda组合行为
allReduce(buf, count, comm, [](void* b, size_t n, Communicator& c) {
    // inline implementation or delegate
});
```

补充场景: Specification模式 — 解决过滤条件的组合爆炸(2个属性需3个函数, 3个属性需7个函数)

```cpp
// 违反OCP: 每增加一种过滤条件组合，就要新加函数
struct ProductFilter {
    static Items byColor(Items items, Color c);
    static Items bySize(Items items, Size s);
    static Items bySizeAndColor(Items items, Size s, Color c);  // 组合爆炸
};

// 遵循OCP: Specification模式 — filter()永远不需要修改
template<typename T>
struct Specification {
    virtual ~Specification() = default;
    virtual bool isSatisfied(const T& item) const = 0;
};

template<typename T>
struct AndSpec : Specification<T> {
    const Specification<T>& a;
    const Specification<T>& b;
    bool isSatisfied(const T& item) const override {
        return a.isSatisfied(item) && b.isSatisfied(item);
    }
};

struct ColorSpec : Specification<Product> {
    Color color;
    bool isSatisfied(const Product& p) const override { return p.color == color; }
};

struct SizeSpec : Specification<Product> {
    Size size;
    bool isSatisfied(const Product& p) const override { return p.size == size; }
};

// 新增过滤条件只需新增Specification子类，不改已有代码
template<typename T>
Items filter(const Items& items, const Specification<T>& spec) {
    Items result;
    for (auto& item : items)
        if (spec.isSatisfied(item)) result.push_back(item);
    return result;
}
```

### 常见误解和陷阱

- 完美OCP不存在: 你不可能在第一次编写时预见所有未来扩展点。OCP是重构方向，不是初始设计目标。当需求冲突迫使重构时，根据当下需求推断未来变化，提取抽象
- 插件架构是OCP的终极形态: Eclipse、Vim的插件系统证明了OCP的强大。核心不依赖插件，插件依赖核心
- 与SRP互补: 为SRP设计的代码天然接近OCP。一个模块只有一个变化理由时，引入新功能就不需要改它
- 过度抽象的代价: 不是每个直接使用另一个类的地方都需要引入抽象。判断标准是: 这个依赖点未来是否真的会有多个实现？
- OCP违反是相对的: 对于特定需求集，代码可能符合OCP；对于另一组需求可能违反。"对什么开放"本身就是一个设计决策
- 两个根本限制: (1) 仍需要某种切换/注册机制来选择具体实现; (2) 系统必须提前设计好支持的扩展方向(新类型 vs 新操作)

---

## 3. 里氏替换原则 (LSP — Liskov Substitution Principle)

### 本质

子类型必须能够替换其基类型而不改变程序的正确性。使用基类指针/引用的代码，在接收任何派生类时都应正常工作。

> If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck. — 但前提是它的行为真的和duck一致

### 违反时的代码信号

- 多态代码块中出现`dynamic_cast`、`typeid`或类型检查 — 最经典的LSP违反信号
- 基类中出现"类型标识位"(`type_`字段)用于区分子类行为
- 子类重写方法时抛出基类未声明的异常
- 子类的前置条件比基类更严格(接受的输入范围更窄)
- 子类的后置条件比基类更宽松(保证的输出范围更大)

### 遵循时的机制

1. 契约一致: 子类的前置条件不能强于基类，后置条件不能弱于基类
2. 行为等价: 基类的所有不变量在子类中必须维持
3. 正确的抽象层次: 如果Square不能完全替换Rectangle(修改width会改height)，它们不应是继承关系。让两者都继承更高的抽象(Shape)
4. 使用工厂模式隐藏构造细节

### C++系统编程典型场景

场景: 通信设备抽象 — 不同传输介质(RDMA、TCP、共享内存)需要统一接口

```cpp
// 违反LSP: 子类行为与基类契约不一致
class Transport {
public:
    virtual void send(const void* buf, size_t len) = 0;
    virtual void recv(void* buf, size_t len) = 0;
    virtual ~Transport() = default;
};

class SharedMemTransport : public Transport {
public:
    void send(const void* buf, size_t len) override {
        if (len > shm_size_) throw std::runtime_error("too large");
        // 基类没有这个限制 — 违反LSP(前置条件更严格)
        std::memcpy(shm_ptr_, buf, len);
    }
    void recv(void* buf, size_t len) override;
};

// 遵循LSP: 在接口层声明约束
class Transport {
public:
    virtual size_t maxMessageSize() const = 0;  // 约束成为契约的一部分
    virtual void send(const void* buf, size_t len) = 0;
    virtual void recv(void* buf, size_t len) = 0;
    virtual ~Transport() = default;
};

class RdmaTransport : public Transport {
public:
    size_t maxMessageSize() const override { return rdmaBufSize_; }
    void send(const void* buf, size_t len) override;
    void recv(void* buf, size_t len) override;
};

class SharedMemTransport : public Transport {
public:
    size_t maxMessageSize() const override { return shmSize_; }
    void send(const void* buf, size_t len) override;
    void recv(void* buf, size_t len) override;
};

// 调用方统一处理，无需类型检查
void broadcast(Transport& t, const void* data, size_t len) {
    if (len > t.maxMessageSize()) {
        // 分块发送 — 统一逻辑，对所有Transport子类有效
    }
    t.send(data, len);
}
```

### LSP违反的四种模式 (Hillel Wayne分类)

1. 前置条件加强: 子类对输入施加比基类更严格的约束(如上例SharedMemTransport的size限制)
2. 后置条件削弱: 子类返回的结果不满足基类承诺的属性
3. 不变量破坏: 子类引入了破坏基类不变量的状态修改(如Square的set_height连带改width)
4. 历史规则违反: 子类引入基类不存在的状态变迁路径

### 常见误解和陷阱

- 用"IS-SUBSTITUTABLE-FOR"替代"IS-A"来判断继承关系。数学上Square IS-A Rectangle，但代码中Square IS-NOT-SUBSTITUTABLE-FOR Rectangle
- LSP的核心不是继承正确，而是抽象正确。正确的抽象层次是松耦合的前提
- 黑盒测试法(Hillel Wayne): "如果X继承Y，则X应该通过Y的所有黑盒测试。"这比形式化的契约规则更直观实用 — 为基类编写测试，所有子类都必须通过
- behavioral subtyping vs syntactic subtyping: 类型签名匹配(协变返回、逆变参数)只是语法层面，LSP要求的是行为层面的可替换性

---

## 4. 接口隔离原则 (ISP — Interface Segregation Principle)

### 本质

客户端不应被迫依赖它用不到的接口方法。接口应该按客户端需求拆分，而非按实现者便利合并。

### 违反时的代码信号

- 实现类中出现空方法体或`throw "not implemented"` — 典型的fat interface
- 一个头文件包含大量函数声明，但大多数使用者只用其中几个
- 修改接口中的某个方法导致不相关的实现类需要重新编译
- 继承一个接口后，发现有些方法根本不适用于当前实现

### 遵循时的机制

1. 按客户端角色拆分接口: 不同的调用者需要不同的接口切面
2. 接口组合: 多重继承(C++中纯虚类的多继承)组合细粒度接口
3. 委托而非继承: 实现类持有各细粒度接口的实例，委托调用

### C++系统编程典型场景

场景: 设备抽象层 — 并非所有设备都支持所有操作

```cpp
// 违反ISP: fat interface
class IDevice {
public:
    virtual void send(const Buffer& buf) = 0;
    virtual void recv(Buffer& buf) = 0;
    virtual void rdmaWrite(const MemRegion& local, const MemRegion& remote) = 0;
    virtual void rdmaRead(const MemRegion& local, const MemRegion& remote) = 0;
    virtual void atomicAdd(uint64_t* remote, uint64_t val) = 0;
    virtual ~IDevice() = default;
};

// TCP设备被迫实现RDMA方法 — 违反ISP
class TcpDevice : public IDevice {
    void rdmaWrite(...) override { throw std::runtime_error("not supported"); }
    void rdmaRead(...) override { throw std::runtime_error("not supported"); }
    void atomicAdd(...) override { throw std::runtime_error("not supported"); }
};

// 遵循ISP: 按能力拆分接口
class ISendRecv {
public:
    virtual void send(const Buffer& buf) = 0;
    virtual void recv(Buffer& buf) = 0;
    virtual ~ISendRecv() = default;
};

class IRdma {
public:
    virtual void rdmaWrite(const MemRegion& local, const MemRegion& remote) = 0;
    virtual void rdmaRead(const MemRegion& local, const MemRegion& remote) = 0;
    virtual ~IRdma() = default;
};

class IAtomic {
public:
    virtual void atomicAdd(uint64_t* remote, uint64_t val) = 0;
    virtual ~IAtomic() = default;
};

class TcpDevice : public ISendRecv { /* 只实现收发 */ };

class RdmaDevice : public ISendRecv, public IRdma, public IAtomic {
    /* 实现全部能力 */
};

// 调用方只依赖它需要的接口
void p2pTransfer(ISendRecv& dev, const Buffer& data) {
    dev.send(data);  // 不关心设备是否支持RDMA
}
```

### 常见误解和陷阱

- ISP关注的不是接口大小，而是客户端是否用到了所有方法。大接口未必违反ISP(如果所有客户端都用全了)，小接口也未必符合ISP(如果抽象本身不正确)
- "堵不如疏": 不要纠结代码是否违反ISP，要考虑你的抽象是否正确。ISP是软件健康度的指标，而非设计时的主要指导力量
- 在C++中，ISP的三大实际效果:
  - 编译速度: fat interface中任何方法签名变更会触发所有派生类重编译，分离后只影响实际使用该接口的类
  - 可读性: `MyDevice : ISendRecv, IRdma`比`MyDevice : IDevice`一目了然地表达了设备能力
  - 可复用性: 消除inadvertent coupling(因fat interface产生的意外耦合)

---

## 5. 依赖反转原则 (DIP — Dependency Inversion Principle)

### 本质

高层模块不应依赖低层模块，两者都应依赖抽象。从行为的角度(做什么)思考设计，而非从构造/实现的角度(怎么做)。

### 违反时的代码信号

- 高层业务逻辑中直接#include低层实现的头文件(如: 算法调度器直接#include具体transport实现)
- 创建对象的代码散落在业务逻辑中(`new ConcreteClass`遍地)
- 修改底层实现导致上层代码需要改动或重新编译
- 依赖图中出现高层 → 低层的单向箭头，没有抽象层

### 遵循时的机制

C++中实现DIP的四种方式:

1. 虚函数(动态多态): 通过抽象基类定义接口，运行时注入实现
2. 模板(静态多态): 编译时注入，零开销但牺牲运行时灵活性
3. `std::function` / 函数对象: 轻量级行为注入
4. Type erasure: 将接口与实现完全解耦(如`std::any`, `std::function`的本质)

### C++系统编程典型场景

场景: 集合通信调度器不应依赖具体的传输实现

```cpp
// 违反DIP: 高层直接依赖低层
#include "rdma_transport.h"  // 高层调度器知道了低层实现细节

class CollectiveScheduler {
    RdmaTransport transport_;  // 直接依赖具体类
public:
    void allReduce(void* buf, size_t count) {
        transport_.send(buf, count);  // 硬编码RDMA
    }
};

// 遵循DIP — 方式1: 虚函数注入
class ITransport {
public:
    virtual void send(const void* buf, size_t len) = 0;
    virtual void recv(void* buf, size_t len) = 0;
    virtual ~ITransport() = default;
};

class CollectiveScheduler {
    ITransport& transport_;  // 依赖抽象
public:
    explicit CollectiveScheduler(ITransport& t) : transport_(t) {}
    void allReduce(void* buf, size_t count) {
        transport_.send(buf, count);  // 不关心是RDMA还是TCP
    }
};
```

```cpp
// 遵循DIP — 方式2: 模板注入(静态多态，零开销)
template<typename Transport>
class CollectiveScheduler {
    Transport transport_;
public:
    void allReduce(void* buf, size_t count) {
        transport_.send(buf, count);
    }
};

// 编译时绑定，没有虚函数开销
CollectiveScheduler<RdmaTransport> scheduler;
```

```cpp
// 遵循DIP — 方式3: std::function(最灵活)
class CollectiveScheduler {
    std::function<void(const void*, size_t)> sendFn_;
public:
    explicit CollectiveScheduler(std::function<void(const void*, size_t)> fn)
        : sendFn_(std::move(fn)) {}
    void allReduce(void* buf, size_t count) {
        sendFn_(buf, count);
    }
};
```

```cpp
// 遵循DIP — 方式4: Type Erasure(最大解耦 — 具体类无需知道抽象接口的存在)
class TransportConcept {                                     // 接口定义
public:
    virtual void send(const void* buf, size_t len) = 0;
    virtual ~TransportConcept() = default;
};

template<typename T>
class TransportModel : public TransportConcept {             // 适配器
    T impl_;
public:
    explicit TransportModel(T impl) : impl_(std::move(impl)) {}
    void send(const void* buf, size_t len) override { impl_.send(buf, len); }
};

class AnyTransport {                                         // 类型擦除包装
    std::unique_ptr<TransportConcept> ptr_;
public:
    template<typename T>
    AnyTransport(T impl) : ptr_(std::make_unique<TransportModel<T>>(std::move(impl))) {}
    void send(const void* buf, size_t len) { ptr_->send(buf, len); }
};

// RdmaTransport不需要继承任何接口 — 真正的解耦
struct RdmaTransport { void send(const void* buf, size_t len); };
struct TcpTransport  { void send(const void* buf, size_t len); };

AnyTransport t = RdmaTransport{};    // 运行时多态，但具体类型不依赖抽象
```

### C++ DIP实现方式性能权衡

| 方式 | vtable开销 | 内联优化 | 编译膨胀 | 解耦程度 | 适用场景 |
|------|-----------|---------|---------|---------|---------|
| 虚函数 | 有 | 不可 | 无 | 中 | 通用场景，类型数适中 |
| 模板 | 无 | 可 | 有 | 低(编译期绑定) | 性能极端敏感 |
| std::function | 有(小对象优化可避免) | 不可 | 无 | 中 | 轻量回调/策略注入 |
| Type Erasure | 有(wrapper层) | 不可 | 有 | 高 | 具体类不能/不应知道接口 |

### DIP/IoC/DI三者关系

- DIP是设计原则(目的): 高层不依赖低层，都依赖抽象
- IoC是设计模式(思路): 控制流反转，框架调用用户代码
- DI是实现手段(工具): 通过构造函数/setter/模板参数把具体实现注入消费者

### 常见误解和陷阱

- DIP不等于"所有东西都要加一层抽象"。只有在需要隔离变化(多个实现/测试替身)时才引入抽象层
- 在性能敏感的系统编程中，优先考虑模板(静态多态)而非虚函数: 零额外开销，且编译器可内联优化。代价是牺牲运行时切换能力
- 依赖方向应始终从不稳定(易变)指向稳定(稳定): 算法变化频率高于传输接口，传输接口变化频率高于传输协议
- DIP的终极目标是"插件架构": 核心不知道插件的存在，插件的所有依赖都指向核心
- YAGNI原则: 虚函数够用就不要为了"高级"而上type erasure。先用最简单的方式实现，复杂度不够时再升级

---

## 6. SOLID原则间的关系

SOLID不是5个独立原则，而是一个从不同视角描述同一件事的整体:

```
SRP: 按变化原因分离 ──┐
                       ├── 高内聚 ──┐
ISP: 按客户端需求分离 ──┘            │
                                     ├── 可维护、可扩展的代码
OCP: 扩展不改旧代码 ──┐             │
                       ├── 低耦合 ──┘
DIP: 依赖抽象不依赖实现 ──┘

LSP: 抽象正确 ── 是上述一切的基础
```

- SRP做好了，OCP自然接近(单一变化原因意味着新功能不影响旧代码)
- DIP是实现OCP的关键手段(通过依赖抽象，让系统对扩展开放)
- ISP是SRP在接口层面的映射(接口也应该有单一职责)
- LSP是所有多态机制正确工作的前提

### SOLID在C++系统编程中的实践优先级

在CANN/hccl/hcomm这类高性能通信库中:

1. SRP(最常用): 模块划分、头文件组织、编译依赖管理
2. DIP(次常用): 传输层抽象、算法策略注入、测试替身
3. OCP(按需): 算法族扩展、设备类型扩展
4. LSP(隐式): 确保所有Transport/Algorithm子类行为一致
5. ISP(偶尔): 当设备能力差异导致fat interface时

---

## 外部参考链接

以下链接在本轮分析中被引用(部分链接可能已失效):

- Uncle Bob原文: http://www.butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod
- Uncle Bob SRP (2014): https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html
- Uncle Bob OCP (2014): https://blog.cleancoder.com/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html
- Uncle Bob OCP (2013): https://blog.cleancoder.com/uncle-bob/2013/03/08/AnOpenAndClosedCase.html
- vishalchovatiya C++ SOLID系列: http://www.vishalchovatiya.com/single-responsibility-principle-in-cpp-solid-as-a-rock/
- Parnas 1972 模块分解论文: https://dl.acm.org/doi/10.1145/361598.361623
- Hillel Wayne LSP分析: https://www.hillelwayne.com/post/lsp/
- SRP的秘密: https://medium.com/hackernoon/the-secret-behind-the-single-responsibility-principle-e2f3692bae25
- OCP解释: https://reflectoring.io/open-closed-principle-explained/
- ISP分析: https://medium.com/hackernoon/interface-segregation-principle-bdf3f94f1d11

## 包的设计原则(补充)

除了类级别的SOLID，Uncle Bob还提出了包级别的6个原则:

内聚性(包里放什么):
- REP(发布复用等价): 复用的粒度等于发布的粒度
- CCP(共同闭包): 因同一原因变化的类打包在一起(SRP的包级版本)
- CRP(共同复用): 经常一起使用的类打包在一起

耦合性(包间关系):
- ADP(无环依赖): 包的依赖图不能有环
- SDP(稳定依赖): 依赖方向朝着稳定性增加的方向
- SAP(稳定抽象): 越稳定的包应该越抽象

这些原则在C++中直接映射到: 头文件组织、静态库/动态库划分、编译单元的依赖管理。
