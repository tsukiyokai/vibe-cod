# Round 4: 模块化 + 接口设计 + 契约 + IoC + 表驱动

## 1. 模块化与信息隐藏

### 1.1 Parnas信息隐藏原则 (1972)

本质: 模块分解的标准应该是"需要隐藏的设计决策"，而不是处理流程。

Parnas用KWIC索引系统对比了两种分解:
- 基于流程图的分解: Input → Circular Shift → Alphabetize → Output，模块共享内存数据结构，任何存储格式变更波及所有模块
- 基于信息隐藏的分解: 每个模块封装一个"可能变更的设计决策"，对外隐藏实现细节

核心准则:
- "It is almost always incorrect to begin the decomposition of a system into modules on the basis of a flowchart."
- 正确做法: 列举"困难的设计决策"和"可能变更的设计决策"，每个决策分配给一个模块负责隐藏
- 模块化的三个收益: 并行开发、变更隔离、可理解性

识别信号 — 模块边界画错了:
- 模块名带"-er"/"-or"后缀(Validator, Sender, Saver) → 过程式思维，应重构为领域实体
- 一个功能变更需要修改多个模块 → 信息未被正确隐藏
- 模块间共享内存/全局数据结构 → 耦合过紧

### 1.2 C语言模块化范式 (云风)

层次分明原则:
- 避免循环依赖 — 分不清谁上谁下是最严重的问题
- 避免越层调用 — A→B→C中A不应直接调用C(基础设施模块除外: 内存、日志、字符串)
- "任何语言上的技巧，都是下策；草率的接口设计往往是日后系统脆弱的根源"

头文件作接口的标准模式:

```c
// a.h — 只声明，不定义
#ifndef _A_h
#define _A_h

struct A;                                   // 不透明类型声明
struct B;                                   // 前向声明，不暴露B的接口

struct A* A_create(void);                   // 构造
void      A_release(struct A *self);        // 析构
void      A_bind(struct A *self, struct B *b);
void      A_commit(struct A *self);         // 实例方法
void      A_update(void);                   // 类方法(无self)
int       A_init(void);                     // 模块初始化

#endif
```

```c
// a.c — 定义与实现
#include "a.h"

struct A {                                  // 结构体定义仅在实现文件中
    int data;
    struct B *b;
};
```

关键规则:
- 不在`a.h`中`#include "b.h"` — 即使A依赖B
- 不主张用typedef减少输入 — "除非有特别理由，都写上struct前缀"
- 第一个参数为self指针(类似C++的this)
- 前缀命名空间(`A_xxxxx`)替代语言级namespace
- 所有私有函数和变量必须用`static`

双向引用的代理类型模式:

```c
// b.h — 下层模块
struct i_A;                                 // 代理类型，仅A模块能构造
void B_set_A(struct B *self, struct i_A *a);

// a.c — 上层模块
void A_bind(struct A *self, struct B *b) {
    B_set_A(b, (struct i_A *)self);         // 只有A能做这个类型转换
}
```

效果: B暴露的接口仅供A使用，其他模块无法获得`struct i_A`类型的值。

### 1.3 CMod四规则 — 类型安全链接

CMod工具对百万行级C代码库的实证: 发现1000+信息隐藏错误、数十个类型错误。

规则1 — 共享头文件: 链接到另一文件中定义的符号时，两个文件都必须include包含该符号类型的头文件。不要用extern绕过。

规则2 — 类型所有权: 每种类型的定义只能由一个文件拥有。头文件中定义 = 透明类型(多模块可见)；源文件中定义 = 不透明类型。

规则3 — 垂直依赖: 头文件必须自包含(self-contained)，不依赖include顺序。

准则2.3 — 解释一致性: 公共头文件的预处理结果在所有include它的编译单元中必须相同。

### 1.4 Pimpl惯用法 (C++的信息隐藏)

问题: 平台特定代码暴露在公共头文件中

```cpp
// BAD: 平台细节泄漏到头文件
class AutoTimer {
    std::string mName;
#ifdef _WIN32
    DWORD mStartTime;
#else
    struct timeval mStartTime;
#endif
};
```

```cpp
// GOOD: Pimpl封装
// autotimer.h
class AutoTimer {
public:
    explicit AutoTimer(const std::string &name);
    ~AutoTimer();
    AutoTimer(const AutoTimer&) = delete;              // 或实现深拷贝
    AutoTimer& operator=(const AutoTimer&) = delete;
private:
    class Impl;
    std::unique_ptr<Impl> mImpl;
};

// autotimer.cpp
class AutoTimer::Impl {
public:
    std::string mName;
#ifdef _WIN32
    DWORD mStartTime;
#else
    struct timeval mStartTime;
#endif
};
```

优势: 信息隐藏、减少编译依赖、二进制兼容性(对象始终sizeof(pointer))
代价: 额外堆分配、间接访问开销、需显式管理拷贝语义

### 1.5 模块初始化管理

C语言没有标准的模块初始化机制。云风提出的mod_using方案:

```c
#define USING(m) if (mod_using(m##_init, #m)) { return 1; }

int foo_init() {
    USING(memory);                          // 声明依赖memory模块
    USING(log);                             // 声明依赖log模块
    // ... foo自己的初始化
    return 0;
}
```

mod_using内部确保不重复初始化且检测循环依赖。宏便于代码扫描器检测遗漏。

### 1.6 导出符号检查

用`nm`验证模块的封装性:

```bash
$ nm module1.o | grep " [A-Z] "            # 大写 = 公共导出
00000014 T module1_add
00000000 D module1_counter
# 所有导出符号都应有模块前缀，否则是信息泄漏
```

---

## 2. 接口设计

### 2.1 命令查询分离 (CQS)

本质: "Asking a question should not change the answer."

规则: 将方法严格分为两类:
- Query: 返回结果，不改变可观察状态(无副作用)
- Command: 改变状态，不返回值(返回void)

C++中的天然支持: `const`修饰符。Query方法标记为const。

CQS与异常安全的深度关联 — pop()反例:

```cpp
// BAD: 违反CQS的pop()
template<typename T>
T Stack<T>::pop() {
    T val = top_;                           // 如果拷贝构造抛异常...
    --size_;                                // 元素已移除，val却丢失
    return val;                             // 违反强异常安全保证
}

// GOOD: 分离command和query
template<typename T>
const T& Stack<T>::top() const;             // query: 查看栈顶
template<typename T>
void Stack<T>::pop();                       // command: 移除栈顶
```

引用Herb Sutter Exceptional C++ Item 10: 即使现代C++ move操作很少抛异常，泛型代码必须支持只有copy语义的类型。某些类型因编译器生成函数规则导致move被意外禁用。

信号 — 何时违反CQS是合理的:
- Fluent Interface的链式调用(setter返回this)
- 原子操作(compare_exchange, fetch_add)
- 迭代器推进(next返回元素)

### 2.2 API设计原则 (综合Qt + Martin Reddy)

六个质量特征:
1. 最小化(minimal): 更少的public成员和类。"有疑问时，留在外面" — 添加比删除容易
2. 完整(complete): 预期功能必须存在
3. 语义清晰: 最小惊讶原则；常见任务简单，罕见任务可行
4. 直觉化: 中等经验用户无需文档即可使用
5. 易记忆: 一致精确的命名，可识别模式，永远不缩写
6. 导向可读代码: 代码读的次数远多于写

信号 — API设计有问题:

```cpp
// BAD: 密码式参数
QSlider *s = new QSlider(12, 18, 3, 13, Qt::Vertical, 0, "volume");

// GOOD: 意图清晰
QSlider *s = new QSlider(Qt::Vertical);
s->setRange(12, 18);
s->setPageStep(3);
s->setValue(13);
```

```cpp
// BAD: bool参数含义不明
widget->repaint(false);                     // 不要重绘？不擦除？

// GOOD: enum替代bool
str.replace("USER", user, Qt::CaseInsensitive);
```

Virtual函数最小化:
- 复杂化bug修复，阻碍编译器内联
- 不能在不破坏二进制兼容性的情况下增删
- 准则: "除非toolkit和主要用户都调用该函数，否则不应是virtual"
- 需要vtable查找(2-3倍于普通调用开销)

降低耦合的具体技术:
- 前向声明优先于#include: "除非需要完整定义，否则用前向声明"
- 非成员非友元函数: "优先声明为非成员非友元而非成员函数" — 提高封装性
- 永远不返回非const指针/引用指向private数据成员
- 永远不让成员变量public — getter/setter提供验证、缓存、日志、同步、不变式维护的扩展点

Const规则:
- const函数接受输入指针应接受const指针
- 一般返回值而非const引用 — 保留未来重构空间
- const函数不应改变可见对象状态

### 2.3 Fluent Interface

本质: API读起来像内部DSL，而非单纯的method chaining。

```cpp
// Fluent Interface示例
auto config = ServerConfig()
    .host("localhost")
    .port(8080)
    .maxConnections(100)
    .timeout(std::chrono::seconds(30));
```

C++实现 — CRTP保持继承链中的类型信息:

```cpp
template <typename Derived>
class BuilderBase {
public:
    Derived& set_name(std::string n) {
        name_ = std::move(n);
        return static_cast<Derived&>(*this);
    }
protected:
    std::string name_;
};

class ConcreteBuilder : public BuilderBase<ConcreteBuilder> {
public:
    ConcreteBuilder& set_port(int p) { port_ = p; return *this; }
private:
    int port_;
};
// 用法: ConcreteBuilder().set_name("srv").set_port(8080);
```

适用场景: 值对象配置、Builder模式、规则声明
代价: 违反CQS(setter返回this)、单个方法脱离上下文无意义、方法级文档难写

### 2.4 正交设计

核心: 站在客户角度设计API，而非从技术实现难易程度出发。

理想目标设定法:
1. 完全不考虑技术实现，思考客户真正需要什么
2. 得到理想API后，寻找一切可能的实现方式
3. 碰到困难时: Try easier(寻找等价但更容易实现的形式) 或 Try harder(走出舒适区)

指导思想: "内防结果出错，外防客户滥用"

### 2.5 幂等性

定义: 同一操作多次执行的影响与一次执行相同。

实现手段:
1. 使用天然幂等的操作(赋值而非累加)
2. 唯一键值/防重表
3. 加锁(乐观锁、分布式锁)
4. source+token验证
5. 状态机(已进入下一状态则拒绝上一状态的变更)

系统编程中的典型场景: 头文件guard宏是最常见的幂等设计。模块初始化函数应该是幂等的。

---

## 3. 契约驱动设计 (DbC)

### 3.1 核心概念

契约 = 前置条件 + 后置条件 + 不变量，语义等同于Hoare三元组。

三个问题:
1. 契约期望什么？(前置条件 — 调用者的义务)
2. 契约保证什么？(后置条件 — 供应商的义务)
3. 契约维护什么？(不变量 — 双方的义务)

### 3.2 防御式 vs 契约式的取舍

关键区分 — error vs exceptional condition:
- Error(bug): 设计/实现缺陷，不可"处理"，只能检测。程序应硬失败(hard fail)
- Exceptional condition: 合法但罕见的运行时情况，需要恢复策略

参数分两类:
- 内部输入(模块间调用): 完全可信，用契约式(断言检测bug)
- 外部输入(用户/网络/文件): 完全不可信，用防御式(检查+错误码/异常)

```cpp
// 内部API — 契约式: 违反即bug，硬失败
void ProcessBuffer(Buffer* buf, size_t offset) {
    REQUIRE(buf != nullptr);
    REQUIRE(offset < buf->size());
    // 专心做正确的事，不需要业务逻辑无关的检查
}

// 外部API — 防御式: 不信任输入
Status ParseUserInput(const char* input, Config* out) {
    if (input == nullptr) return Status::InvalidArg;
    if (strlen(input) > MAX_INPUT_LEN) return Status::TooLong;
    // 验证通过后调用内部API
}
```

原则: "好的代码核心是处理正确的事情，而不是一堆错误检查"。不要害怕崩溃 — 崩溃促使及时发现和解决错误。try-catch隐藏错误是在挖坑。

### 3.3 嵌入式/系统编程的Assert框架 (Barr Group)

标准C的assert()对嵌入式不适用(print+exit)。语义化宏:

```c
#ifdef NASSERT                              // 与NDEBUG解耦，独立控制
#define ASSERT(ignore_)  ((void)0)
#define ALLEGE(test_)    ((void)(test_))    // 即使禁用仍求值表达式
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

设计要点:
- `DEFINE_THIS_FILE`避免`__FILE__`字符串在每个assert处重复，节省ROM
- `NASSERT`与`NDEBUG`解耦 — IDE切换Release时不会自动禁用
- `ALLEGE()`解决有副作用的表达式问题: `ALLEGE(x = read())` 在NASSERT下仍执行赋值
- `onAssert__()`回调: 嵌入式中通常禁中断→fail-safe→系统reset→存文件名/行号到非易失存储

实践规则:
1. Preemptive > Defensive: 主动寻找可以assert的假设
2. 发布代码保留assertion: "练习时穿救生衣，真正上场时却不穿" — C.A.R. Hoare
3. 断言副作用警告: ASSERT()内的表达式在NASSERT模式下不执行
4. 强前置条件消除大量defensive检查代码

### 3.4 C++中的契约支持

C++20 Contract提案(已推迟到C++26):

```cpp
int mul(int x, int y)
    [[expects: x > 0]]                     // 前置条件
    [[expects default: y > 0]]
    [[ensures audit res: res > 0]] {       // 后置条件(audit级别)
    return x * y;
}
```

当前实践 — GSL的Expects/Ensures:

```cpp
#include <gsl/gsl>

double sqrt(double x) {
    Expects(x >= 0);                        // 前置条件
    // ...
    Ensures(result >= 0);                   // 后置条件
    return result;
}
```

后置条件防止优化器删除关键操作:

```cpp
void f() {
    char buffer[MAX];
    memset(buffer, 0, sizeof(buffer));
    Ensures(buffer[0] == 0);                // 阻止optimizer删除memset
}
```

### 3.5 继承中的契约

子类继承父类的契约，但:
- 前置条件可以更弱(weaker) — 接受更多输入
- 后置条件可以更强(stronger) — 保证更多输出
- 不变量可以更强 — 维护更多约束

这直接对应LSP的机械化检验。违反此规则 = 违反LSP。

### 3.6 契约与测试的关系

契约不替代测试，但:
- 契约提供所有可能案例的抽象表示，测试只覆盖特定案例
- 引入DbC可减少测试用例数量
- 开发流程: 1.写契约 → 2.写测试+写代码(交织进行)
- 契约是自动生成测试套件的基础

断言监控的调试层级:
1. 不检查(生产环境可选)
2. 只检查前置条件(默认)
3. 前置条件+后置条件
4. 上述+类不变量
5. 所有断言

---

## 4. IoC与依赖注入

### 4.1 核心原则

"配置与使用的分离是根本原则，具体用DI还是Service Locator是次要的。" — Fowler

IoC的本质: 框架控制流程，组件提供实现。"应用程序员调用utils如同娶妻，使用框架如同做上门女婿。"

### 4.2 三种注入形式在C++中的映射

构造函数注入(推荐) — 天然适配RAII:

```cpp
class MessageProcessor {
public:
    // 构造时获取所有依赖，对象始终处于合法状态
    explicit MessageProcessor(
        std::unique_ptr<ITransport> transport,
        std::unique_ptr<ISerializer> serializer)
        : transport_(std::move(transport))
        , serializer_(std::move(serializer)) {}

    void Process(const Message& msg) {
        auto data = serializer_->Serialize(msg);
        transport_->Send(data);
    }

private:
    std::unique_ptr<ITransport> transport_;
    std::unique_ptr<ISerializer> serializer_;
};
```

Setter注入 — 参数过多或需要延迟配置时:

```cpp
class Pipeline {
public:
    void SetEncoder(std::unique_ptr<IEncoder> enc) { encoder_ = std::move(enc); }
    void SetDecoder(std::unique_ptr<IDecoder> dec) { decoder_ = std::move(dec); }
    // 风险: 调用Process时encoder_/decoder_可能为空
};
```

模板注入(C++特有) — 编译期绑定，零运行时开销:

```cpp
template <typename Transport, typename Serializer>
class MessageProcessor {
public:
    void Process(const Message& msg) {
        auto data = Serializer::Serialize(msg);
        Transport::Send(data);
    }
};
// 实例化: MessageProcessor<TcpTransport, ProtobufSerializer>
```

### 4.3 DI vs Service Locator

选DI: 编写供他人使用的库 — 不知道用户用什么Locator
选Service Locator: 应用内部代码，部署环境已知

C++系统编程中的Service Locator:

```cpp
class ServiceLocator {
public:
    static void Provide(std::shared_ptr<ILogger> logger) {
        logger_ = std::move(logger);
    }
    static ILogger& GetLogger() {
        assert(logger_);                    // 契约: 必须先Provide
        return *logger_;
    }
private:
    static std::shared_ptr<ILogger> logger_;
};
```

### 4.4 C++中DI的实现方式对比

| 方式 | 运行时开销 | 灵活性 | 适用场景 |
|------|-----------|--------|---------|
| 虚函数(接口类) | vtable查找 | 运行时替换 | 插件架构、测试mock |
| 模板参数 | 零 | 编译期绑定 | 性能敏感路径 |
| std::function | 间接调用+可能堆分配 | 运行时替换 | 回调、策略 |
| Type Erasure | 间接调用 | 运行时替换+值语义 | 需要值语义的多态 |

---

## 5. 表驱动法

### 5.1 核心思想

"数据压倒一切。如果选择了正确的数据结构并把一切组织得井井有条，正确的算法就不言自明。" — Rob Pike

本质: 数据是易变的，逻辑是稳定的。将代码的复杂度转移到数据中去。

Unix哲学中的两条原则:
- 分离原则: 策略同机制分离，接口同引擎分离
- 表示原则: 把知识叠入数据以求逻辑质朴而健壮

### 5.2 三种查表方式

直接查找 — O(1):

```c
char n2c = "0123456789ABCDEF"[num];         // 数字转十六进制字符
```

索引查找 — 间接映射:

```c
// 消息码 → 处理函数
typedef void (*MsgHandler)(const void* payload);

typedef struct {
    uint32_t    msgCode;
    MsgHandler  handler;
} MsgDispatchEntry;

static const MsgDispatchEntry gDispatchTable[] = {
    { MSG_INIT,     HandleInit },
    { MSG_DATA,     HandleData },
    { MSG_SHUTDOWN, HandleShutdown },
};
```

分段查找 — 区间映射:

```c
typedef struct {
    double      rangeLimit;
    const char* grade;
} GradeMap;

static const GradeMap gGradeMap[] = {
    {50.0,  "Fail"},
    {60.0,  "Pass"},
    {70.0,  "Credit"},
    {80.0,  "Distinction"},
    {100.0, "High Distinction"},
};

static const char* EvaluateGrade(double score) {
    for (size_t i = 0; i < ARRAY_SIZE(gGradeMap); i++) {
        if (score < gGradeMap[i].rangeLimit)
            return gGradeMap[i].grade;
    }
    return gGradeMap[0].grade;
}
```

### 5.3 信号 — 何时用表驱动替代if-else

- 3个以上的if-else/switch-case分支处理相似逻辑
- 状态机的状态转移
- 命令模式的命令码分发
- 消息处理的消息码路由
- 配置和流程需要分离时

### 5.4 数据驱动编程的层次

- 函数级: 查找表替代条件分支(本节示例)
- 程序级: 表驱动状态机
- 系统级: DSL、配置文件驱动的行为

### 5.5 函数指针的风险

"函数已退化成指针，指针不携带类型信息" — 参数和返回值类型不受编译器检查。

C++中的改进: 用`std::function`或模板替代裸函数指针，获得类型安全。

---

## 6. 信号槽

### 6.1 本质

信号槽是Observer模式的C++实现，提供对象间事件驱动通信，无需知道彼此的实现细节。

### 6.2 Pure C++实现 (无Qt依赖)

```cpp
#include <functional>
#include <map>

template <typename... Args>
class Signal {
public:
    // 连接任意callable，返回连接ID
    int connect(std::function<void(Args...)> const& slot) const {
        slots_.insert(std::make_pair(++current_id_, slot));
        return current_id_;
    }

    // 连接成员函数
    template <typename T>
    int connect_member(T* inst, void (T::*func)(Args...)) {
        return connect([=](Args... args) { (inst->*func)(args...); });
    }

    void disconnect(int id) const { slots_.erase(id); }
    void disconnect_all() const   { slots_.clear(); }

    void emit(Args... args) {
        for (auto const& [id, slot] : slots_) {
            slot(args...);
        }
    }

private:
    mutable std::map<int, std::function<void(Args...)>> slots_;
    mutable int current_id_{0};
};
```

使用:

```cpp
class Button {
public:
    Signal<> on_click;                      // 无参信号
};

class Dialog {
public:
    void close() { /* ... */ }
};

Button btn;
Dialog dlg;
btn.on_click.connect_member(&dlg, &Dialog::close);
btn.on_click.emit();                        // 触发dlg.close()
```

### 6.3 设计决策与权衡

- `mutable`成员: connect/disconnect是逻辑const操作
- `std::map<int, ...>`: 整数ID用于精确断开连接
- 变参模板: signal可传递任意类型组合

已知限制:
- 非线程安全 — 需加锁保护slots_
- 不能在slot回调内部disconnect当前signal的连接(迭代中修改容器)
- 生命周期管理需自行处理 — 对象销毁后指针悬挂

增强方向:
- 线程安全: 加mutex或用lock-free数据结构
- 生命周期: std::weak_ptr追踪slot对象是否存活
- 遍历安全: emit时先拷贝slots_快照(参见Round 3 Observer安全增强)

### 6.4 信号槽 vs 直接回调

信号槽的优势:
1. 一对多: 一个signal可连接多个slot
2. 线程协同: Qt的信号槽支持跨线程队列投递
3. 可读性: `connect(sender, signal, receiver, slot)`比嵌套回调更清晰
4. 可动态连接/断开

直接回调更适合: 一对一、性能敏感、生命周期简单的场景。

---

## 横向总结: 可操作的决策指引

### 模块边界怎么画

1. 列举系统中"可能变更的设计决策"
2. 每个决策分配给一个模块隐藏
3. 检验: 模块名是领域名词而非动词(-er/-or)
4. 检验: 单个需求变更只影响一个模块

### 接口暴露什么

1. "有疑问时，留在外面" — 先不暴露，需要时再加
2. 永远不暴露成员变量
3. virtual函数谨慎添加 — 无法在不破坏兼容性的情况下删除
4. 用enum替代bool参数
5. 一般返回值而非const引用

### 契约放哪里

1. 外部输入(用户/网络): 防御式，返回错误码
2. 内部接口(模块间): 契约式，REQUIRE/ENSURE/INVARIANT
3. 不变量: 复杂状态的类必须有，在每个public方法入口和出口检查
4. 发布代码保留assertion — 检测bug的价值远大于性能开销

### 依赖怎么注入

1. 默认用构造函数注入(适配RAII)
2. 性能敏感路径用模板参数(零开销)
3. 需要运行时替换用虚函数接口
4. 回调/策略用std::function

### 何时用表驱动

1. 3+分支处理相似逻辑
2. 策略和机制需要分离
3. 数据比逻辑更容易变化
4. 消息分发、状态转移、命令路由
