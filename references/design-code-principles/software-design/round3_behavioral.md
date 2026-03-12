# Round 3: 行为型设计模式 — C++系统编程视角

## 实用性分级总览

| 模式 | 实用性 | 系统编程场景 |
|------|--------|-------------|
| State Machine (状态机) | 极高 | 协议栈、设备管理、任务生命周期 |
| Observer (观察者) | 极高 | 事件通知、模块解耦、状态同步 |
| Strategy (策略) | 高 | 算法选择、行为注入、回调 |
| Template Method (模板方法) | 高 | 框架骨架、流程统一、钩子扩展 |
| Iterator (迭代器) | 高 | 容器遍历、C接口设计 |
| Chain of Responsibility (职责链) | 高 | 请求分发、校验链、事件处理 |
| Visitor (访客) | 中 | 异构集合操作、双重派发、AST遍历 |
| Command (命令) | 中 | 操作封装、宏命令、延迟执行 |
| Mediator (中介) | 中 | 消息总线、模块间通信 |
| Memento (备忘录) | 低 | 状态快照/回滚(系统编程中少见) |

---

## 极高实用性

### 1. State Machine (状态机)

本质: 用显式状态 + 事件驱动替代散落在代码各处的条件逻辑，让对象行为取决于"我在哪个状态"而非"我经历了什么标志位组合"。

信号 — 代码中出现这些特征说明需要状态机:
- 多个bool标志位相互配合决定行为(`isRunning && !isPaused && isReady`)
- switch/if-else链中反复检查同一组条件
- 函数行为取决于"上次做了什么"
- 注释里出现"在XX状态下才能YY"

#### 1.1 实现层级: 从简单到复杂

层级一: 状态转移表(表驱动)

最简洁的实现，适合状态少、转移规则固定的场景。

```cpp
// Mealy machine: 输出取决于当前状态 + 输入事件
enum State { IDLE, STARTED, STOPPED, STATE_COUNT };
enum Event { EVT_START, EVT_STOP, EVENT_COUNT };

struct Transition {
    State next;
    void (*action)(void* ctx);
};

// 整个状态机行为浓缩在一张表里
static const Transition table[STATE_COUNT][EVENT_COUNT] = {
    /*             EVT_START                    EVT_STOP              */
    /* IDLE    */ {{STARTED, on_start},         {IDLE,    nullptr}},
    /* STARTED */ {{STARTED, nullptr},          {STOPPED, on_stop}},
    /* STOPPED */ {{STARTED, on_start},         {IDLE,    on_reset}},
};

void dispatch(State& state, Event evt, void* ctx) {
    auto& t = table[state][evt];
    if (t.action) t.action(ctx);
    state = t.next;
}
```

优点: 状态转移一目了然，新增状态/事件只需扩展表格。
适用: 通信协议状态(如简单握手)、设备开关控制。

层级二: 函数指针分发

每个状态是一个函数，状态机通过函数指针分发事件。

```c
// 状态即函数 — 指针指向谁就是什么状态
typedef void (*StateHandler)(Fsm* me, Event const* e);

typedef struct Fsm {
    StateHandler state;    // 当前状态
    // ... 扩展状态变量
} Fsm;

void fsm_dispatch(Fsm* me, Event const* e) {
    me->state(me, e);     // 直接分发，效果等同于虚函数多态
}

void fsm_tran(Fsm* me, StateHandler target) {
    me->state = target;   // 状态转移 = 换指针
}

// 使用: 每个状态函数内部用switch处理事件
void state_idle(Fsm* me, Event const* e) {
    switch (e->sig) {
        case START_SIG:
            do_start_action(me);
            fsm_tran(me, &state_running);
            break;
        // 未处理的事件自然忽略
    }
}
```

优点: 状态函数自包含所有该状态的行为，代码结构直接映射状态图。
来源: Miro Samek的QP框架核心技术，在嵌入式领域广泛使用。

层级三: 层次状态机(HSM)

解决平坦状态机的"状态爆炸"问题 — 当多个状态共享相同的事件处理时，用嵌套替代重复。

```
// 层次关系: 子状态未处理的事件会上升到父状态
//
// operational (handles OFF event)
//   +-- vehiclesEnabled (handles TIMEOUT)
//   |     +-- vehiclesGreen
//   |     +-- vehiclesYellow
//   +-- pedestriansEnabled (handles TIMEOUT)
//         +-- pedestriansWalk
//         +-- pedestriansFlash
```

核心机制:
- 状态函数返回父状态指针(表示"我没处理，交给上级")或NULL(表示"已处理")
- entry/exit action沿层次链执行，类似构造/析构函数的调用顺序
- 子状态只定义与父状态的差异("programming by difference")

```c
// HSM状态函数签名: 返回父状态(未处理)或NULL(已处理)
typedef StateHandler (*StateHandler)(Fsm* me, Event const* e);

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

Liskov替换原则应用于状态: 子状态不能违反父状态的语义契约。`vehiclesEnabled`的子状态不能启用行人通行，否则破坏了父状态"车辆通行中"的不变量。

#### 1.2 实战案例: 密钥更新状态机

来自Notion笔记的真实案例，融合单例 + 观察者 + 状态机:

```
IDLE --[update_request]--> UPDATING --[success]--> COOLING_DOWN --[expired]--> IDLE
                               |
                               +--[failure]---> IDLE
```

设计要点:
- 状态(`IDLE`/`UPDATING`/`COOLING_DOWN`)封装了复杂的资格检查逻辑
- 冷却期用时间窗口模式实现，避免了在状态机内部管理定时器细节
- entry/exit action保证状态变量一致性(`isUpdating`, `isInCooling`)

#### 1.3 陷阱与决策

陷阱一: guard condition滥用。

> Guard condition的滥用是状态机架构腐化的首要原因 — 因为它重新引入了状态机本应消除的复杂条件逻辑。(Miro Samek)

如果一个转移需要检查3个以上guard，考虑是否应该增加状态来替代guard。

陷阱二: 将状态机与流程图混淆。

状态机是事件驱动的(被动等待事件)，流程图是转换驱动的(完成一步自动进入下一步)。状态函数不应返回值 — 状态机必须始终准备接受事件。

选择哪个层级?
- 状态 <5，转移规则固定 → 表驱动
- 状态 5-15，需要清晰的状态行为封装 → 函数指针
- 状态 >15，有明显的层次共性 → HSM

---

### 2. Observer (观察者)

本质: 让对象在状态变化时自动通知所有关注方，而不需要知道谁在关注。

信号 — 代码中出现这些特征说明需要观察者:
- 一个对象改变后需要手动调用多个其他对象的更新方法
- 注释写着"别忘了在这里也更新XX"
- 模块间存在"你改了A就得改B"的隐式约定

#### 2.1 C++实现: 从简单到安全

最小实现:

```cpp
class Observable;

class Observer {
public:
    virtual ~Observer() = default;
    virtual void update(Observable& source) = 0;
};

class Observable {
    std::set<Observer*> observers_;
public:
    void attach(Observer* o)  { observers_.insert(o); }
    void detach(Observer* o)  { observers_.erase(o); }
protected:
    void notify() {
        for (auto* o : observers_) o->update(*this);
    }
};
```

问题: 这个最小实现有三个致命缺陷 —

1. 生命周期悬挂: Observer被销毁后Observable仍持有其指针
2. 通知期间修改: notify()遍历中有Observer执行detach()导致迭代器失效
3. 同步阻塞: 某个Observer的update()耗时长会阻塞整个通知链

#### 2.2 安全增强(来自ACCU三部曲)

生命周期安全 — 双向跟踪:

```cpp
class SafeObserver {
    std::set<Observable*> attached_to_;
public:
    virtual ~SafeObserver() {
        // 析构时自动断开所有连接
        for (auto* obs : attached_to_) obs->detach(this);
    }
    void attach_to(Observable* obs) {
        obs->attach(this);
        attached_to_.insert(obs);
    }
};
```

Observable的析构也要通知所有Observer(通过`deleted()`回调)，实现双向自动清理。

通知安全 — 迭代期间修改:

```cpp
void notify() {
    // 复制一份列表再遍历，允许回调中执行detach
    auto snapshot = observers_;
    for (auto* o : snapshot) {
        if (observers_.count(o)) {  // 双重检查: 是否在遍历期间被移除
            o->update(*this);
        }
    }
}
```

#### 2.3 嵌入式环境: ETL模板方案

Embedded Template Library提供零堆分配的观察者实现:

```cpp
// 编译期固定最大观察者数
class SensorDriver : public etl::observable<SensorObserver, 4> {
    void on_data_ready(int val) {
        notify_observers(val);  // 按类型分发到不同的notification()重载
    }
};

// 观察者可选择性实现不同类型的通知
class Logger : public etl::observer<int, ErrorCode> {
    void notification(int val)       { log_value(val); }
    void notification(ErrorCode err) { log_error(err); }
};
```

特点: 固定容量(无堆分配)、多类型通知(模板参数列表)、未实现的通知类型自动忽略。

#### 2.4 性能陷阱: 模板膨胀

ACCU文章中的真实教训: 使用`std::set`和`std::vector`作为观察者列表的模板实现，导致库从5MB膨胀到16MB。

解决方案: 内部用`void*`列表 + 类型安全的外部接口。在嵌入式场景中尤其要注意模板实例化的代码体积代价。

#### 2.5 推模型 vs 拉模型

- 推模型: `notify(data)` — Observable把数据推给Observer
- 拉模型: `notify()` → Observer回调`source.getState()` — Observer按需获取

推模型耦合度高但效率好(避免回调查询)。拉模型灵活但多了一次间接调用。系统编程中常用推模型(减少延迟)。

---

## 高实用性

### 3. Strategy (策略)

本质: 将算法从方法变成对象，使算法可在运行时替换。

信号:
- 函数内部有大段if/else选择不同的处理逻辑
- 同一个类的不同实例需要不同的行为
- 注释写着"将来可能换成XX算法"

机制: Context持有抽象策略引用，客户端注入具体策略。

```cpp
// 现代C++: std::function替代虚函数继承体系
class Sorter {
    std::function<void(std::vector<int>&)> strategy_;
public:
    void set_strategy(std::function<void(std::vector<int>&)> s) {
        strategy_ = std::move(s);
    }
    void sort(std::vector<int>& data) { strategy_(data); }
};

// 使用: lambda即策略
Sorter sorter;
sorter.set_strategy([](auto& v) { std::sort(v.begin(), v.end()); });
// 运行时切换
if (data.size() < 16)
    sorter.set_strategy([](auto& v) { insertion_sort(v); });
```

策略 vs 虚函数继承:
- 策略是运行时委托(has-a)，模板方法是编译时继承(is-a)
- 策略可以在对象生命周期内切换，继承不行
- 策略用`std::function`/lambda时零类定义开销

C语言实现: 函数指针 + 枚举查表

```c
typedef int (*CompareFunc)(const void*, const void*);

// 策略表: 枚举 -> 函数指针
static const CompareFunc compare_strategies[] = {
    [SORT_ASC]  = compare_ascending,
    [SORT_DESC] = compare_descending,
    [SORT_ABS]  = compare_absolute,
};

void sort_data(Data* data, SortOrder order) {
    qsort(data->items, data->count, sizeof(Item), compare_strategies[order]);
}
```

本质上策略模式等价于回调。在函数式编程中，高阶函数自然地实现了策略模式。C++中有了lambda后，很多简单的策略场景不再需要完整的类层次结构。

---

### 4. Template Method (模板方法)

本质: 基类定义算法骨架(步骤顺序不可变)，子类填充可变步骤。

注意: 这里的"模板"是设计模式术语，与C++的template语法无关。

信号:
- 多个类有相似的处理流程，只是某些步骤不同
- 复制粘贴修改一两行的代码重复
- 流程控制逻辑散布在多个子类中

机制:

```cpp
class DataProcessor {
public:
    // 模板方法: 定义不可变的流程骨架
    void process() {
        validate();      // 可被子类覆盖
        transform();     // 可被子类覆盖
        output();        // 可被子类覆盖
        log_complete();  // 不可覆盖的公共步骤
    }
protected:
    virtual void validate()  = 0;  // 必须由子类实现
    virtual void transform() = 0;
    virtual void output()    = 0;
private:
    void log_complete() { /* 所有子类共享 */ }
};
```

核心要点:
- 模板方法本身应该是non-virtual的(NVI — Non-Virtual Interface)
- 步骤函数是protected virtual
- 控制反转: 不是子类调用框架，而是框架调用子类

CRTP静态替代:

当不需要运行时多态时，CRTP消除虚函数开销:

```cpp
template <typename Impl>
class DataProcessor {
public:
    void process() {
        static_cast<Impl*>(this)->validate();
        static_cast<Impl*>(this)->transform();
    }
};

class CsvProcessor : public DataProcessor<CsvProcessor> {
public:  // 注意: 不是virtual
    void validate()  { /* CSV校验 */ }
    void transform() { /* CSV转换 */ }
};
```

策略 vs 模板方法: 策略用委托(has-a, 运行时切换)，模板方法用继承(is-a, 编译时固定)。如果算法骨架不变只是步骤变化，用模板方法；如果整个算法可能被替换，用策略。

---

### 5. Iterator (迭代器)

本质: 提供统一接口遍历聚合对象的元素，不暴露底层实现。

C++标准库已提供完善的迭代器体系。这里聚焦C语言中的迭代器接口设计 — 在系统编程(尤其是暴露C API的库)中极为常见。

#### 5.1 C迭代器三种模式

模式一: Index Access(索引访问)

```c
// 接口: 用户按索引获取元素
int    collection_count(Collection* c);
Item*  collection_get(Collection* c, int index);

// 使用
for (int i = 0; i < collection_count(c); i++) {
    Item* item = collection_get(c, i);
    process(item);
}
```

优点: 简单直观，支持随机访问。
缺点: 遍历期间增删元素导致索引失效；对链表等结构O(n)访问。

模式二: Cursor Iterator(游标迭代器/first-done-next)

```c
// 接口: 三件套 -> 自然映射到for循环
Item* collection_first(Collection* c);
bool  collection_done(Collection* c, Item* cursor);
Item* collection_next(Collection* c, Item* cursor);

// 使用 — 不依赖索引，对任何数据结构高效
for (Item* it = collection_first(c);
     !collection_done(c, it);
     it = collection_next(c, it)) {
    process(it);
}
```

优点: 隐藏内部结构，链表和数组同样高效。
缺点: 遍历期间修改集合仍有风险。
适用: 大部分C API场景的默认选择。

模式三: Callback Iterator(回调迭代器)

```c
// 接口: 对每个元素调用用户函数
typedef void (*ItemCallback)(Item* item, void* user_data);
void collection_foreach(Collection* c, ItemCallback cb, void* user_data);

// 使用
void print_item(Item* item, void* data) {
    printf("%s\n", item->name);
}
collection_foreach(c, print_item, NULL);
```

优点: 实现方可以在遍历期间持锁保证一致性；用户不接触内部状态。
缺点: 需要额外函数和void*传递上下文；无法中途break(除非约定返回值)。
适用: 需要保证遍历原子性、或遍历逻辑复杂(嵌套、递归)的场景。

#### 5.2 选择依据

| 场景 | 推荐模式 |
|------|---------|
| 简单数组/缓冲区，需要随机访问 | Index Access |
| 通用容器遍历，对外API | Cursor (first/done/next) |
| 需要遍历原子性(持锁遍历) | Callback |
| 遍历中有复杂嵌套逻辑 | Callback |

---

### 6. Chain of Responsibility (职责链)

本质: 用对象链替代级联if-else，请求沿链传递直到被某个处理者处理。

信号:
- 长串if-else/switch处理不同类型的请求
- 请求需要经过多级校验/审批
- 处理者需要动态增减

```cpp
class Handler {
    Handler* next_ = nullptr;
protected:
    virtual bool can_handle(const Request& req) = 0;
    virtual void process(const Request& req) = 0;
public:
    void set_next(Handler* h) { next_ = h; }

    void handle(const Request& req) {
        if (can_handle(req))
            process(req);
        else if (next_)
            next_->handle(req);
        // else: 请求未被处理(考虑是否需要默认处理)
    }
};
```

系统编程典型应用:
- 网络包处理管线(各层协议依次解析)
- 权限校验链(先检查黑名单 → 再检查白名单 → 再检查角色)
- 日志分级(debug handler → info handler → error handler)

何时不用: 如果条件分支固定且简单(如只有3-4个case且不会扩展)，plain if-else比对象链更清晰。职责链的价值在于运行时可组合和可扩展。

---

## 中等实用性

### 7. Visitor (访客)

本质: 在不修改类层次结构的前提下，向其添加新的多态操作。本质上是"外部虚函数" — 实现双重派发。

信号:
- 需要对异构对象集合执行多种不同操作
- 想在不改变类接口的情况下添加新行为
- 代码中出现大量`dynamic_cast`或`typeid`判断

何时使用: 元素类型稳定但操作频繁变化。
何时不用: 元素类型频繁变化(每加一个类型所有Visitor都要改)、数据同质、操作少。

双重派发机制:

```cpp
// 第一次分发: 虚函数解析到正确的Element子类
// 第二次分发: Element子类调用Visitor的对应重载
class Visitor {
public:
    virtual void visit(Circle& c) = 0;
    virtual void visit(Square& s) = 0;
};

class Shape {
public:
    virtual void accept(Visitor& v) = 0;  // 第一次派发
};

class Circle : public Shape {
public:
    void accept(Visitor& v) override {
        v.visit(*this);  // this是Circle& -> 第二次派发到visit(Circle&)
    }
};
```

减少样板代码的宏:

```cpp
#define DECLARE_VISITABLE() \
    void accept(Visitor& v) override { v.visit(*this); }

class Circle : public Shape { DECLARE_VISITABLE() /* ... */ };
class Square : public Shape { DECLARE_VISITABLE() /* ... */ };
```

现代C++替代: `std::variant` + `std::visit`

当类型集合在编译期已知时，variant消除了虚函数和accept样板:

```cpp
using Shape = std::variant<Circle, Square, Triangle>;

// 用overloaded lambda替代Visitor类
auto area = std::visit(overloaded{
    [](Circle& c)   { return 3.14 * c.r * c.r; },
    [](Square& s)   { return s.side * s.side; },
    [](Triangle& t) { return 0.5 * t.base * t.h; },
}, shape);
```

CRTP泛化Visitor(ETL风格):

```cpp
// 被访问者通过CRTP混入accept能力
template <typename Impl>
class VisitableShape : public IShape {
public:
    template <typename V>
    void accept(V&& v) { v.visit(static_cast<Impl&>(*this)); }
};

class Circle : public VisitableShape<Circle> { /* ... */ };
```

无需RTTI，编译期类型安全，适合嵌入式场景。

---

### 8. Command (命令)

本质: 将操作封装为对象，使操作可以排队、撤销、组合。

信号:
- 需要支持undo/redo
- 操作需要排队或延迟执行
- 需要将多个操作组合为宏

```cpp
class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
};

// 宏命令 = Command + Composite
class MacroCommand : public Command {
    std::vector<std::unique_ptr<Command>> cmds_;
public:
    void add(std::unique_ptr<Command> c) { cmds_.push_back(std::move(c)); }
    void execute() override {
        for (auto& c : cmds_) c->execute();
    }
};
```

现代C++中，简单的Command场景直接用`std::function<void()>`即可，不需要继承体系。只有需要undo、序列化、或复杂组合时才需要完整的Command类。

---

### 9. Mediator (中介)

本质: 引入中间人协调多个对象的交互，避免对象间直接引用形成网状耦合。

信号:
- 多个模块相互调用形成复杂依赖图
- 新增一个模块需要修改多个已有模块

一句话机制: 中介持有所有参与者的引用，参与者只持有中介引用。通信从`A->B, A->C, B->C`变成`A->M->B, A->M->C`。

系统编程中的体现: 消息总线、事件分发器。当Observer模式的"一对多"不够用，需要"多对多"协调时，Mediator是自然的演进。

---

## 低实用性

### 10. Memento (备忘录)

在不暴露内部状态的情况下捕获对象快照，以便后续恢复。

系统编程中较少直接使用。如果需要状态快照/回滚，通常通过序列化、checkpoint机制或copy-on-write实现，而非经典的Memento类结构。

---

## 跨模式洞察

### 行为型模式与函数式编程的关系

C++引入lambda后，多个行为型模式的经典OO实现变得不必要:

| 模式 | 函数式等价 |
|------|-----------|
| Strategy | `std::function` / lambda |
| Command | `std::function<void()>` |
| Observer(简单) | callback list |
| Template Method | 传入step函数 |
| Iterator | range-based for / generator |

原则: 如果行为型模式的核心只是"把行为参数化"，用lambda。只有当需要额外结构(undo、组合、生命周期管理、类型安全分发)时才用完整的模式类。

### 状态机 + 观察者 = 事件驱动架构

在系统编程中，状态机和观察者经常协同工作:
- 观察者负责事件的收集和分发
- 状态机负责根据当前状态决定如何响应事件
- 命令模式封装事件为可排队的对象

这三者组合构成了嵌入式系统和通信协议栈的基础架构模式。

### 模式选择速查

- "行为随状态变化" → State Machine
- "变化时通知他人" → Observer
- "算法可替换" → Strategy (或直接lambda)
- "流程固定步骤可变" → Template Method
- "条件链可扩展" → Chain of Responsibility
- "操作可撤销/排队" → Command
- "异构集合多种操作" → Visitor (或`std::variant`+`std::visit`)

---

## 外部链接抓取记录

成功:
- barrgroup.com: 层次状态机介绍、状态机编码、事件驱动状态机 (3篇)
- accu.org: Observer实现三部曲 (Part 1/2/3)
- etlcpp.com: Observer教程、Visitor教程
- cs.yale.edu: C迭代器模式 (first/done/next, callback)
- sii.pl: 职责链模式处理复杂校验
- state-machine.com: QP/C++ 框架概述

失败:
- codeproject.com: State Machine Design in C/C++ (2篇, 连接中断)
- embedded.com: State-oriented programming (超时)
- cs.oberlin.edu: OO vs FP (404)
- wiki.c2.com: State-Class Duality (JS渲染页面无法抓取)
- adamtornhill.com: Patterns in C - STATE/STRATEGY (PDF二进制)
