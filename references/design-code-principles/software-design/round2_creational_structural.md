# Round 2: 创建型 + 结构型设计模式 — C++系统编程实战分析

> 源材料: Notion设计模式笔记(15个文件) + 外部资源(design-patterns-for-humans, unforgettable-factory, Linux kernel OO patterns, SourceMaking, C++17设计模式, Alternatives to Singletons等)

## 实用性分级标准

- 高: 在C++系统编程(hccl/hcomm/CANN)中频繁出现，值得详细掌握
- 中: 偶尔用到或间接体现，了解核心思想即可
- 低: 极少直接使用，一句话带过

---

## 一、创建型模式

### 1. Singleton (单例) — 高实用性

本质: 保证一个类只有一个实例，并提供全局访问点。

#### 违反信号 (什么代码说明你需要单例)

- 多个模块各自创建同一个"管理器"/"配置"/"日志"对象，状态不同步
- 全局变量散落在多个翻译单元，初始化顺序不确定导致crash
- 构造函数执行昂贵初始化(打开设备、读配置文件)，被多次执行浪费资源

#### 机制

C++11后唯一推荐写法 — Meyer's Singleton:

```cpp
class DeviceManager {
public:
    static DeviceManager& GetInstance() {
        static DeviceManager instance;    // C++11保证线程安全
        return instance;
    }

    DeviceManager(const DeviceManager&) = delete;             // public=delete
    DeviceManager& operator=(const DeviceManager&) = delete;  // 比private更明确

    void OpenDevice(int id) { /* ... */ }

private:
    DeviceManager() { /* 昂贵初始化 */ }
    ~DeviceManager() = default;
};
```

要点:
1. 局部static变量，C++11保证线程安全，不需要DCLP或mutex
2. `public = delete`优于`private`声明 — 编译器先检查可访问性再检查弃置，错误信息更明确(Scott Meyers, Effective Modern C++)
3. 析构函数protected或private，防止外部delete
4. AOSP从2017年起禁止用mutex实现单例，推荐scoped static initialization

CRTP泛化版(系统中单例多时减少重复代码):

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
    ~Singleton() = default;   // 非虚，protected防止基类指针delete
};

// 使用:
class ConfigManager : public Singleton<ConfigManager> {
    friend Singleton<ConfigManager>;  // 允许基类访问私有构造
    // public方法...
private:
    ConfigManager() { /* ... */ }
    ~ConfigManager() = default;
};
```

CRTP版的设计决策:
- 基类构造/析构: protected(不能private，否则派生类无法构造)
- 派生类构造/析构: private + friend声明(最严格)
- 基类析构非虚: 不会通过基类指针delete派生对象，virtual无意义
- 派生类无需重复`= delete`: C++11基类弃置后派生类自动弃置

宏展开方案(Apollo Cyber RT风格):

```cpp
#define DECLARE_SINGLETON(classname)                               \
public:                                                            \
  static classname* Instance(bool create_if_needed = true) {       \
    static classname* instance = nullptr;                          \
    if (!instance && create_if_needed) {                           \
      static std::once_flag flag;                                  \
      std::call_once(flag, [&] {                                   \
        instance = new (std::nothrow) classname();                 \
      });                                                          \
    }                                                              \
    return instance;                                               \
  }                                                                \
private:                                                           \
  classname();                                                     \
  DISALLOW_COPY_AND_ASSIGN(classname)
```

#### 典型场景

- 设备驱动管理器、日志系统、配置文件管理、串口资源管理
- 工厂类本身通常实现为单例
- 标准库: `std::cin`, `std::cout`, `std::cerr`

#### 陷阱与争议

1. Static Initialization Order Fiasco: 不同翻译单元的全局static初始化顺序不确定。Meyer's Singleton用局部static解决了初始化顺序，但析构顺序仍不确定。多个单例析构时互相依赖会出问题(Modern C++ Design的Phoenix Singleton是完整解决方案)
2. 单例是反模式的争论: 核心问题是隐式全局状态使测试困难。实用态度 — 如果对象语义上确实全局唯一(设备驱动、硬件资源管理器)，单例合理；如果仅仅为了"方便访问"，考虑依赖注入
3. 替代方案(Bob Schmidt): Monostate NVI — 用NVI + shared_ptr<abstract_base>提供全局访问但保持可替换性:

```cpp
class mono_nvi {
public:
    explicit mono_nvi(std::shared_ptr<abstract_base> p) { mp = p; }
    mono_nvi() { }
    void foo() { mp->foo(); }  // inline转发，编译器优化为单次虚调用
private:
    static std::shared_ptr<abstract_base> mp;
};
```

---

### 2. Factory Method (工厂方法) — 高实用性

本质: 定义创建对象的接口，让子类决定实例化哪个类。`new`是具体的，工厂是抽象的。

#### 违反信号

- 代码中大量`new ConcreteClass`散布在业务逻辑中
- switch/if-else链根据类型字符串创建不同对象
- 新增产品类型需要修改已有代码(违反OCP)
- 客户端代码同时知道抽象接口和具体实现类

#### 机制

核心思想: 请求一个对象和创建一个对象之间有区别。`new`总是创建具体对象，而工厂方法封装了这种创建，允许请求者与创建行为解耦。

一看到`new`就要想到"具体" — 代码绑定具体类会导致脆弱和缺乏弹性。

手动注册工厂(实用且简洁):

```cpp
class ITransport {
public:
    virtual void Send(const void* data, size_t len) = 0;
    virtual ~ITransport() = default;
};

using CreateFn = std::function<std::unique_ptr<ITransport>()>;

class TransportFactory {
public:
    static TransportFactory& GetInstance() {
        static TransportFactory inst;
        return inst;
    }

    void Register(const std::string& name, CreateFn fn) {
        creators_[name] = std::move(fn);
    }

    std::unique_ptr<ITransport> Create(const std::string& name) {
        auto it = creators_.find(name);
        if (it == creators_.end()) return nullptr;
        return it->second();
    }

private:
    std::unordered_map<std::string, CreateFn> creators_;
};

// 注册: 每新增一个Transport，加一行
TransportFactory::GetInstance().Register("tcp", [] {
    return std::make_unique<TcpTransport>();
});
```

自动注册工厂(Unforgettable Factory — 零手动注册):

```cpp
template <class Base, class... Args>
class Factory {
public:
    template <class T>
    struct Registrar : Base {
        // 自动注册: 利用静态变量初始化在main之前执行
        static bool registered;
        static bool registerT() {
            Factory::data().emplace(
                typeid(T).name(),
                [](Args... args) -> std::unique_ptr<Base> {
                    return std::make_unique<T>(std::forward<Args>(args)...);
                });
            return true;
        }
        // 防止链接器优化掉未引用的静态变量
        Registrar() { (void)registered; }
    };

    static std::unique_ptr<Base> Make(const std::string& name, Args&&... args) {
        auto& m = data();
        auto it = m.find(name);
        return it == m.end() ? nullptr : it->second(std::forward<Args>(args)...);
    }

private:
    using FuncType = std::function<std::unique_ptr<Base>(Args...)>;
    static auto& data() {
        static std::unordered_map<std::string, FuncType> s;
        return s;
    }
};

template <class Base, class... Args>
template <class T>
bool Factory<Base, Args...>::Registrar<T>::registered =
    Factory<Base, Args...>::Registrar<T>::registerT();

// 使用: 继承Registrar而非Base，自动注册
class TcpTransport : public Factory<ITransport>::Registrar<TcpTransport> {
    void Send(const void* data, size_t len) override { /* ... */ }
};
// TcpTransport在程序启动时自动注册到Factory中
```

关键技术:
- CRTP + 静态变量初始化trick: `static bool registered`在main前执行
- Passkey idiom: Base构造函数接受私有Key对象，防止绕过Registrar直接继承Base
- 构造函数中`(void)registered`强制实例化静态变量，防止链接器优化掉

#### 典型场景

- 通信传输层: 根据配置创建TCP/RDMA/共享内存Transport
- 算子工厂: 根据设备类型创建CPU/NPU/GPU算子
- 序列化/反序列化: 根据类型标识重建对象

#### 设计演化路径

工厂方法 → 抽象工厂 → 原型(灵活性递增，复杂度递增)

---

### 3. Abstract Factory (抽象工厂) — 中实用性

本质: 提供创建一族相关对象的接口，无需指定具体类。工厂方法创建一个产品，抽象工厂创建一族产品。

信号: 系统需要在多个产品族之间切换(如跨平台UI: Gtk vs Qt，加密套件: AES-128 vs AES-256)，且同一族的产品必须一起使用。

```cpp
// 抽象工厂: 创建一族加密相关对象
class ICryptoFactory {
public:
    virtual std::unique_ptr<ICipher> CreateCipher() = 0;
    virtual std::unique_ptr<IKeyMaterial> CreateKeyMaterial() = 0;
    virtual ~ICryptoFactory() = default;
};

class Aes128Factory : public ICryptoFactory {
    std::unique_ptr<ICipher> CreateCipher() override {
        return std::make_unique<Aes128Cipher>();
    }
    std::unique_ptr<IKeyMaterial> CreateKeyMaterial() override {
        return std::make_unique<Aes128KeyMaterial>();
    }
};
```

与工厂方法的区别: 抽象工厂创建的产品来自同一个产品族(强制约束一致性)，工厂方法创建单个产品。

---

### 4. Builder (建造者) — 中实用性

本质: 将复杂对象的构造过程与表现分离，相同构造流程产出不同结果。

信号:
- 构造函数参数过多(>4个)，部分可选
- 某些参数组合非法
- 对象构造有固定步骤但步骤实现可变

GoF Builder(Director + Builder分离):

```cpp
// 场景: 构建通信消息，步骤固定，内容因协议而异
class IMessageBuilder {
public:
    virtual void BuildHeader() = 0;
    virtual void BuildPayload(const void* data, size_t len) = 0;
    virtual void BuildChecksum() = 0;
    virtual ~IMessageBuilder() = default;
};

class MessageDirector {
    IMessageBuilder& builder_;
public:
    explicit MessageDirector(IMessageBuilder& b) : builder_(b) {}
    void Construct(const void* data, size_t len) {
        builder_.BuildHeader();
        builder_.BuildPayload(data, len);
        builder_.BuildChecksum();
    }
};
```

Bloch Builder(非GoF，Fluent Interface):

注意: 这不是GoF Builder。由Joshua Bloch提出，目的是简化多参数对象构造。C++因为有默认参数，对此需求不如Java强烈。

```cpp
class ConnectionConfig {
public:
    class Builder {
    public:
        Builder& SetHost(std::string h) { cfg_.host = std::move(h); return *this; }
        Builder& SetPort(int p) { cfg_.port = p; return *this; }
        Builder& SetTimeout(int ms) { cfg_.timeout_ms = ms; return *this; }
        ConnectionConfig Build() { return std::move(cfg_); }
    private:
        ConnectionConfig cfg_;
    };
private:
    std::string host;
    int port = 0;
    int timeout_ms = 5000;
    friend class Builder;
};

// 使用:
auto cfg = ConnectionConfig::Builder()
    .SetHost("10.0.0.1")
    .SetPort(8080)
    .SetTimeout(3000)
    .Build();
```

---

### 5. Prototype (原型) — 低实用性

本质: `virtual T* clone() const;` — 通过复制已有对象创建新对象，即虚构造函数。

在C++系统编程中极少直接使用。当你只有基类指针但需要派生类副本时才需要:

```cpp
class Shape {
public:
    virtual std::unique_ptr<Shape> Clone() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
    std::unique_ptr<Shape> Clone() const override {
        return std::make_unique<Circle>(*this);  // 调用拷贝构造
    }
};
```

CRTP自动化clone(避免每个派生类重复写):

```cpp
template <typename Base, typename Derived>
class Cloneable : public Base {
public:
    std::unique_ptr<Base> Clone() const override {
        return std::make_unique<Derived>(static_cast<const Derived&>(*this));
    }
};

class Circle : public Cloneable<Shape, Circle> { /* ... */ };
```

适用场景: 对象创建成本高(需要从网络/数据库/文件读取数据)时，克隆比重新创建快。资源池(数据库连接池、线程池)可以看作原型模式的应用。

---

## 二、结构型模式

### 6. Adapter (适配器) — 高实用性

本质: 将一个类的接口转换成客户端期望的另一个接口。在不修改现有代码的前提下拓展其行为。

#### 违反信号

- 引入第三方库，其接口与系统约定不一致
- 旧模块接口不满足新需求，但不能/不该修改旧代码
- 多个功能相似但接口不同的类需要统一使用方式

#### 机制

两种形态:

对象适配器(组合，运行时，更灵活，推荐):

```cpp
// 系统期望的接口
class ICommChannel {
public:
    virtual int Send(const void* buf, size_t len) = 0;
    virtual ~ICommChannel() = default;
};

// 第三方库的现有类，接口不兼容
class LegacySocket {
public:
    int TransmitData(const char* data, int size) { /* ... */ }
};

// 对象适配器: 包装LegacySocket
class SocketAdapter : public ICommChannel {
    LegacySocket& socket_;
public:
    explicit SocketAdapter(LegacySocket& s) : socket_(s) {}
    int Send(const void* buf, size_t len) override {
        return socket_.TransmitData(
            static_cast<const char*>(buf), static_cast<int>(len));
    }
};
```

类适配器(多继承，编译时):

```cpp
class SocketAdapter : public ICommChannel, private LegacySocket {
public:
    int Send(const void* buf, size_t len) override {
        return TransmitData(static_cast<const char*>(buf),
                            static_cast<int>(len));
    }
};
```

选择标准:
- 对象适配器: 能适配Adaptee及其所有子类，运行时可替换
- 类适配器: 可重定义Adaptee行为(因为是子类)，无额外指针开销

#### 典型场景

- 封装C库为C++接口(系统编程中极常见)
- 统一不同硬件厂商的驱动接口
- 将旧协议栈适配到新框架

---

### 7. Facade (外观) — 高实用性

本质: 为复杂子系统提供简化的高层接口。迪米特法则(最少知道原则)的直接体现。

#### 违反信号

- 客户端代码需要了解子系统的多个类才能完成一个操作
- 多个调用者都在重复相同的子系统调用序列
- 子系统内部变更导致大量外部代码修改

#### 机制

```cpp
// 子系统的复杂组件
class DeviceDriver { /* ... */ };
class MemoryAllocator { /* ... */ };
class StreamManager { /* ... */ };

// Facade: 一键式接口
class CommEngine {
public:
    // 客户端只需调用这一个方法
    Status Initialize(const Config& cfg) {
        auto ret = driver_.Open(cfg.device_id);
        if (ret != OK) return ret;
        ret = allocator_.AllocateBuffers(cfg.buffer_count, cfg.buffer_size);
        if (ret != OK) return ret;
        return stream_.Setup(driver_, allocator_);
    }

    Status SendData(const void* data, size_t len) {
        auto buf = allocator_.GetBuffer();
        std::memcpy(buf.ptr, data, len);
        return stream_.Enqueue(buf);
    }

private:
    DeviceDriver driver_;
    MemoryAllocator allocator_;
    StreamManager stream_;
};
```

核心要点:
- 子系统对Facade一无所知(无反向依赖)
- Facade不禁止直接访问子系统(需要定制化的高级用户仍可直接用)
- 分层架构中每层入口点就是Facade

---

### 8. Proxy (代理) — 高实用性

本质: 用代理对象控制对真实对象的访问，可附加额外行为(延迟初始化、访问控制、日志审计)。

#### 违反信号

- 资源初始化昂贵但可能不被使用(需要lazy loading)
- 需要在调用前进行权限检查、参数校验
- 需要对调用进行计数、计时、日志记录
- 真实对象在远端，需要本地代理处理通信细节

#### 机制

```cpp
class ICompute {
public:
    virtual Status Execute(const Tensor& input, Tensor& output) = 0;
    virtual ~ICompute() = default;
};

class RealCompute : public ICompute {
public:
    Status Execute(const Tensor& input, Tensor& output) override {
        // 真正的计算逻辑
    }
};

// 代理: 添加lazy init + 计时
class ComputeProxy : public ICompute {
public:
    Status Execute(const Tensor& input, Tensor& output) override {
        LazyInit();
        auto start = std::chrono::steady_clock::now();
        auto ret = real_->Execute(input, output);
        auto elapsed = std::chrono::steady_clock::now() - start;
        LOG("Execute took %lld us",
            std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count());
        return ret;
    }

private:
    void LazyInit() {
        if (!real_) {
            real_ = std::make_unique<RealCompute>();
        }
    }
    std::unique_ptr<RealCompute> real_;
};
```

代理类型:
- 虚代理(Virtual Proxy): 延迟创建昂贵对象
- 保护代理(Protection Proxy): 权限控制
- 远程代理(Remote Proxy): 隐藏网络通信细节
- 智能引用(Smart Reference): `std::shared_ptr`本身就是引用计数代理

---

### 9. Composite (组合) — 高实用性

本质: 将对象组合成树结构，让客户端统一处理单个对象和组合对象。"部分即整体"。

#### 违反信号

- 代码中对单个对象和对象集合做相同操作但用了不同代码路径
- 树形/层次结构中需要递归遍历
- 需要表示部分-整体关系

#### 机制

```cpp
// 场景: 通信拓扑中的节点层次
class ITopologyNode {
public:
    virtual void Execute(const Task& task) = 0;
    virtual ~ITopologyNode() = default;
};

// 叶节点: 单个设备
class DeviceNode : public ITopologyNode {
    int device_id_;
public:
    explicit DeviceNode(int id) : device_id_(id) {}
    void Execute(const Task& task) override {
        // 在单个设备上执行
    }
};

// 组合节点: 设备组
class DeviceGroup : public ITopologyNode {
    std::string name_;
    std::vector<std::unique_ptr<ITopologyNode>> children_;
public:
    explicit DeviceGroup(std::string name) : name_(std::move(name)) {}

    void Add(std::unique_ptr<ITopologyNode> node) {
        children_.push_back(std::move(node));
    }

    void Execute(const Task& task) override {
        for (auto& child : children_) {
            child->Execute(task);  // 递归: 无论child是叶还是组合
        }
    }
};

// 使用: 客户端不区分单设备和设备组
auto group = std::make_unique<DeviceGroup>("ring0");
group->Add(std::make_unique<DeviceNode>(0));
group->Add(std::make_unique<DeviceNode>(1));
group->Execute(task);  // 和单个DeviceNode调用方式相同
```

已知应用: GUI控件树、AST(抽象语法树)、XML/JSON文档模型、文件系统。

常与其他模式组合: Composite + Iterator(遍历树)、Composite + Visitor(对树操作)、Composite + Flyweight(共享叶节点)。

---

### 10. Decorator (装饰器) — 中实用性

本质: 动态地给对象附加额外职责，比子类继承更灵活。递归/链表结构。

信号: 需要透明地给对象增加功能，且功能组合可能很多(子类爆炸)。

动态装饰器(经典GoF):

```cpp
class IStream {
public:
    virtual void Write(const void* data, size_t len) = 0;
    virtual ~IStream() = default;
};

class FileStream : public IStream {
    void Write(const void* data, size_t len) override { /* 写文件 */ }
};

// 装饰器: 加密后再写
class EncryptedStream : public IStream {
    std::shared_ptr<IStream> wrapped_;
public:
    explicit EncryptedStream(std::shared_ptr<IStream> s) : wrapped_(std::move(s)) {}
    void Write(const void* data, size_t len) override {
        auto encrypted = Encrypt(data, len);
        wrapped_->Write(encrypted.data(), encrypted.size());
    }
};

// 可以层层装饰
auto stream = std::make_shared<EncryptedStream>(
    std::make_shared<BufferedStream>(
        std::make_shared<FileStream>("log.bin")));
```

静态装饰器(CRTP Mixin — 编译期，零额外开销):

```cpp
struct Circle {
    float radius = 10.0f;
    std::string GetName() const {
        return "circle(r=" + std::to_string(radius) + ")";
    }
};

template <typename T>
struct ColoredShape : public T {
    std::string color;
    explicit ColoredShape(std::string c) : color(std::move(c)) {}
    std::string GetName() const {
        return T::GetName() + " [color=" + color + "]";
    }
};

using RedCircle = ColoredShape<Circle>;  // 编译期组合
```

CRTP装饰器优势: 零虚函数开销，编译期类型检查，可叠加(`ColoredShape<SizedShape<Circle>>`)。

---

### 11. Bridge (桥接) — 中实用性

本质: 将抽象与实现解耦，使两者可以独立变化。防止笛卡尔积类爆炸。

信号: 一个类有两个或更多独立变化的维度，且各维度需要独立扩展。

```cpp
// 维度1: 抽象 — 消息类型
class IMessage {
protected:
    class ITransport& transport_;  // 桥: 持有实现的引用
public:
    explicit IMessage(ITransport& t) : transport_(t) {}
    virtual void Send() = 0;
    virtual ~IMessage() = default;
};

// 维度2: 实现 — 传输方式
class ITransport {
public:
    virtual void Transmit(const void* data, size_t len) = 0;
    virtual ~ITransport() = default;
};

// 两个维度独立扩展，M种消息 * N种传输 只需 M+N 个类而非 M*N
```

Pimpl是桥接模式的退化形式: 将类的实现细节隔离到单独的Impl类中，头文件只暴露公共接口。编译防火墙。

```cpp
// widget.h
class Widget {
public:
    Widget();
    ~Widget();
    void DoSomething();
private:
    struct Impl;
    std::unique_ptr<Impl> pimpl_;
};
```

---

### 12. Flyweight (享元) — 中实用性

本质: 通过共享细粒度对象的公共部分来减少内存使用。区分内在状态(可共享、不变)和外在状态(不可共享、由客户端传入)。

信号: 系统中存在大量相似对象，对象的大部分状态可以外部化。

```cpp
// 享元工厂 + 池
class SymbolTable {
public:
    // 返回共享的字符串引用，避免重复存储
    const std::string& Intern(const std::string& str) {
        auto [it, _] = pool_.insert(str);
        return *it;
    }
private:
    std::set<std::string> pool_;
};
```

经典案例: 活字印刷(内在=字形，外在=位置)。`std::string`的SSO(Small String Optimization)和`boost::flyweight`是标准库级别的享元实现。

享元工厂通常实现为单例。策略/状态对象常作为享元实现(无状态，可共享)。

---

## 三、OOP in C — 特别关注

> C语言也有其优雅之处，切忌削足适履。

### 为什么重要

Linux内核、嵌入式系统、CANN底层C代码都在用C实现面向对象设计。理解这些技术对于:
- 阅读内核/驱动代码
- 在C/C++混合项目中保持设计一致性
- 理解C++虚函数的底层机制

### 封装 in C

```c
// 不透明指针 (Opaque Pointer / Abstract Data Type)
// device.h — 公开接口
typedef struct Device Device;              // 前向声明，用户看不到内部
Device* device_create(int id);
void device_destroy(Device* dev);
int device_send(Device* dev, const void* data, size_t len);

// device.c — 实现细节隐藏
struct Device {
    int id;
    int fd;
    void* internal_buffer;
};
```

`static`函数/变量限制在翻译单元内 — 最基础的封装。

### 接口 in C — 函数指针结构体

```c
// 接口 = 一组函数指针
struct transport_ops {
    int (*open)(struct transport* t, const char* addr);
    int (*send)(struct transport* t, const void* data, size_t len);
    int (*recv)(struct transport* t, void* buf, size_t len);
    void (*close)(struct transport* t);
};
```

### 继承 in C — 结构体内嵌

```c
// "基类"
struct transport {
    const struct transport_ops* ops;   // vtable指针
    int ref_count;
};

// "派生类": 把基类作为第一个成员
struct tcp_transport {
    struct transport base;             // 继承(内存布局兼容)
    int sockfd;
    struct sockaddr_in addr;
};

// 向上转型: 安全，指向子类的指针与指向基类的指针天然兼容
struct transport* t = (struct transport*)tcp;

// 向下转型: container_of宏(Linux内核)
#define container_of(ptr, type, member) \
    ((type*)((char*)(ptr) - offsetof(type, member)))

struct tcp_transport* tcp = container_of(t, struct tcp_transport, base);
```

### 多态 in C — 虚函数表

```c
// 虚函数包装器(non-virtual interface wrapper)
static inline int transport_send(struct transport* t,
                                  const void* data, size_t len) {
    return t->ops->send(t, data, len);  // 通过vtable分发
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
    .recv  = tcp_recv,
    .close = tcp_close,
};

struct tcp_transport* tcp_transport_create(void) {
    struct tcp_transport* tcp = calloc(1, sizeof(*tcp));
    tcp->base.ops = &tcp_ops;     // 绑定vtable
    return tcp;
}
```

### Linux内核中的OO模式(来自lwn.net分析)

1. Vtable命名惯例: `<object>_operations`或`<object>_ops`(如`file_operations`, `inode_operations`)
2. 可空函数指针: 允许渐进式开发，调用者检查NULL并提供默认行为。但更好的做法是提供显式的默认函数
3. Mixin模式: 与主vtable平行的额外ops结构体，为对象附加可选能力(如`export_operations`为文件系统添加NFS导出能力)
4. 嵌入式函数指针: 对于方法少或需要个体定制的对象，函数指针直接放在结构体中而非独立vtable

核心洞见: 即使不用面向对象语言，内核中也充满了对象、类(表现为vtable)和mixin。语言特性只提供便利，不是实现OOP的必要条件。

---

## 四、模式间关系

### 创建型模式演化路径

```
简单工厂 → 工厂方法 → 抽象工厂 → 原型
(switch/case)  (多态)    (产品族)   (克隆)
        复杂度递增，灵活性递增
```

- 工厂方法模式中的工厂类通常实现为Singleton
- Builder专注分步构造，Factory专注一步创建
- Prototype在C++中可用工厂+反射替代

### 结构型模式对比

| 模式 | 关键词 | 核心机制 |
|------|--------|----------|
| Adapter | 兼容 | 转换已有接口 |
| Bridge | 解耦 | 抽象与实现分离 |
| Composite | 树 | 统一处理个体与组合 |
| Decorator | 增强 | 递归包装附加功能 |
| Facade | 简化 | 高层接口隐藏复杂性 |
| Flyweight | 共享 | 共享不变状态节省内存 |
| Proxy | 控制 | 代理对象控制访问 |

### 防爆炸三件套

状态空间爆炸(M个维度 * N个变化)时:
- Bridge: 分离两个独立变化维度 → M+N个类
- Decorator: 功能可叠加组合 → M+N个类
- Composite: 部分与整体统一处理 → 递归结构

---

## 五、外部链接抓取记录

成功抓取:
1. design-patterns-for-humans (GitHub) — 所有模式的简洁真实世界类比
2. nirfriedman.com/unforgettable-factory — C++自动注册工厂完整实现
3. lwn.net/Articles/444910 — Linux内核OO设计模式深度分析
4. hedzr.com C++17设计模式 — 现代C++模式实现概览
5. sourcemaking.com/factory_method — 工厂方法深度分析
6. accu.org Alternatives to Singletons — Monostate NVI替代方案

抓取失败:
- vishalchovatiya.com/composite-design-pattern-in-modern-cpp — 返回404
- adamtornhill.com Patterns in C (Reactor PDF) — PDF无法解析为文本
