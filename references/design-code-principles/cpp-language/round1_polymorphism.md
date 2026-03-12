# Round 1: 多态与接口实现机制

## 来源

- $MAIN L937-L1386 (范式: OOP + 模板元 + 函数式)
- $SUB: 虚函数调用机制、crtp、type erasure、rtti、mixin、delegates、static_cast、dynamic_cast
- WebFetch: Eli Bendersky CRTP文章、FluentCpp Concepts替代CRTP、simoncoenen Delegates、
  Microsoft Proxy库、lwithers协变返回类型、Abseil Tips #94/#99/#198/#218/#234

---

## 1. 虚函数调用机制

信号: 通过基类指针/引用调用函数时，需要理解将调用哪个版本。

机制:
- 虚函数调用 → 按底层对象的实际类型解析(动态绑定)
- 非虚函数调用 → 按顶层指针/引用的声明类型解析(静态绑定)
- `::` 作用域限定符可抑制虚调用: `pBase->Account::PrintBalance()` 强制调基类版本
- 覆盖函数始终为虚，即使不写virtual关键字(隐式继承虚性)
- 基类虚函数必须被定义，除非声明为纯虚(`= 0`)

```cpp
class Base {
public:
    virtual void NameOf()    { cout << "Base\n"; }
    void InvokingClass()     { cout << "Base\n"; }   // 非虚
};
class Derived : public Base {
public:
    void NameOf()            { cout << "Derived\n"; } // 隐式virtual
    void InvokingClass()     { cout << "Derived\n"; } // 隐藏，非覆盖
};

Base* p = new Derived;
p->NameOf();         // "Derived" — 虚函数看底层对象
p->InvokingClass();  // "Base"    — 非虚函数看顶层指针
```

陷阱:
- 派生类隐藏(hide)同名非虚函数 ≠ 覆盖(override)虚函数，两者行为完全不同
- 忘记在基类中将函数声明为virtual → 永远调基类版本，debug困难
- 构造函数/析构函数中调虚函数 → 不发生动态绑定(此时对象类型尚未完成或已部分销毁)

---

## 2. 继承体系决策

### 2.1 私有继承 vs 组合

信号: 需要复用另一个类的实现(而非接口)时。

机制:
- 私有继承 = is-implemented-in-terms-of，不是is-a
- 基类的public/protected成员变为派生类的private成员 → 基类方法不暴露为派生类接口
- 组合(composition)是实现has-a的默认选择

决策准则: "Use composition when you can, private inheritance when you have to."

何时不得不用私有继承:
- 需要访问protected成员
- 需要覆盖虚函数(如定制一个timer的回调)
- 利用空基类优化(EBO)节省空间

```cpp
// 组合(默认选择)
class Car {
    Engine eng;  // Car has-a Engine
public:
    void start() { eng.start(); }
};

// 私有继承(特殊情况)
class Car : private Engine {  // 也是has-a，但可访问protected
public:
    void start() { Engine::start(); }
};
```

陷阱:
- 私有继承可能引发多重继承问题
- 外部代码无法将私有派生类转换为基类(`eat(student)` 编译失败)
- "私有继承在软件设计过程中毫无意义，只有在软件实现过程中才有意义"

### 2.2 多态4条准则

信号: 设计类继承体系时。

| 准则 | 含义 |
|------|------|
| 应有虚函数皆为虚 | 通过基类接口执行的操作都应声明为virtual |
| 非叶子类应抽象 | 中间层基类应该是抽象类，不应直接实例化 |
| 不要从具体类继承 | 只从抽象类继承，保持LSP |
| 析构函数二选一 | public virtual 或 protected non-virtual |

析构函数准则详解:
- public virtual: 允许通过基类指针delete → 正确调用派生类析构
- protected non-virtual: 禁止通过基类指针delete → 编译期阻止错误使用

### 2.3 final优化

信号: 类不再被继承，或虚函数不再被覆盖。

机制: 编译器看到final后可进行去虚化(devirtualization)，将虚调用优化为直接调用。

```cpp
class Leaf final : public Base {
    void process() override { /* ... */ }  // 编译器可内联此调用
};
```

---

## 3. 协变返回类型(Covariant Return Types)

信号: 覆盖虚函数时，返回类型是基类返回类型的派生类指针/引用。
典型场景: 原型模式的clone()方法。

机制:
- 如果基类虚函数返回`Base*`，派生类覆盖可以返回`Derived*`
- 条件: `Derived`必须public继承自`Base`
- 通过基类指针调用时返回类型自动向上转型为`Base*`
- 通过派生类直接调用时返回`Derived*`，免去static_cast

```cpp
class Base {
public:
    virtual Base* clone() const { return new Base(*this); }
};

class Derived : public Base {
public:
    Derived* clone() const override {  // 协变: Derived* 而非 Base*
        return new Derived(*this);
    }
};

// 使用
Derived d;
Derived* copy = d.clone();         // 直接得到Derived*，无需cast
Base* bCopy = d.clone();           // 也合法，自动向上转型
```

陷阱:
- 协变仅对裸指针和引用有效，不适用于智能指针
  (`unique_ptr<Derived>`和`unique_ptr<Base>`是不相关的类型)
- 若需要智能指针+协变: 用非虚public包装函数 + private虚函数返回裸指针

---

## 4. CRTP(Curiously Recurring Template Pattern)

信号: 需要编译期多态(避免虚函数开销)，或需要给不同类型混入相同的扩展功能。

机制:
- 派生类继承自以自身为模板参数的基类: `class D : public Base<D>`
- 基类通过`static_cast<Derived*>(this)`访问派生类成员
- 模板方法只在被调用时才实例化(惰性实例化) → 基类可以"假装"调用派生类尚未定义的方法

```cpp
template <typename Derived>
struct Base {
    void interface() {
        static_cast<Derived*>(this)->impl();  // 编译期绑定
    }
};

struct D1 : Base<D1> {
    void impl() { cout << "D1\n"; }
};

template <typename T>
void call(Base<T>& b) { b.interface(); }  // 静态多态
```

6大应用:

| 应用 | 说明 |
|------|------|
| 静态多态 | 免虚函数表的多态分发 |
| 运算符扩展(Mixin) | 只实现`<`，自动获得`==` `!=` `>` `<=` `>=` |
| 对象计数 | `Counter<T>`为每个T维护独立计数器 |
| 链式多态 | 方法返回自身引用，支持fluent API |
| Facade模式 | 基类根据派生类的小接口定义大public接口 |
| Template Method | 算法骨架在基类，具体步骤由派生类实现 |

运算符扩展的关键逻辑(用`<`推导其他比较):
```
a == b ↔ !(a < b) && !(b < a)
a != b ↔ (a < b) || (b < a)
a > b  ↔ b < a
a <= b ↔ !(b < a)
a >= b ↔ !(a < b)
```

C++20替代: Concepts可以直接约束接口，消除CRTP的名字冲突和间接层:
```cpp
template <typename T>
concept Drawable = requires(T t, Canvas& c) {
    t.draw(c);
};

template <Drawable T>
void render(T& shape) { shape.draw(canvas); }
// 无需继承Base<T>，无需static_cast
```

陷阱:
- CRTP的`Base<D1>`和`Base<D2>`是不同类型 → 不能放入同一容器(无运行时多态)
- static_cast的安全性由用户保证(如果D没有从Base<D>继承就UB)
- 多个CRTP基类 → 多重继承变丑("the only remaining choice is macros")

---

## 5. 模板名字查找两阶段(CRTP核心陷阱)

信号: 在模板基类的派生类中调用基类方法，编译报"no declaration available"。

机制:
- 第一阶段(定义时): 查找与模板参数无关的名字(无依赖名字)
- 第二阶段(实例化时): 查找与模板参数有关的名字(有依赖名字)
- 问题: 在`Base2<T> : Base<T>`中直接调`Method()`，编译器在第一阶段找不到它(因为`Base<T>`是依赖类型，此时尚未实例化)

```cpp
template<typename T>
class Base {
public:
    virtual const string& Method() const = 0;
};

template<typename T>
class Base2 : public Base<T> {
public:
    void Func() const {
        cout << Method() << endl;  // 错误! Method是无依赖名字，第一阶段找不到
    }
};
```

正确解法(二选一):
```cpp
// 解法1: using声明引入名字
using Base<T>::Method;

// 解法2: 显式this使之成为依赖名字
this->Method()
```

错误解法:
```cpp
Base<T>::Method()  // 编译通过，但变成静态绑定!
                   // 如果Method是纯虚的 → 运行时崩溃
```

为什么`Base<T>::Method()`是错的:
- 指定了类型后，调用变成静态绑定(等效于`::` 抑制虚调用)
- 如果Base中该函数是纯虚的，没有定义 → 运行时行为未定义

---

## 6. Type Erasure

信号: 需要在运行时以统一方式操作不同类型的对象，但不想要/不能用继承。

机制:
- 类型擦除 = OOP(运行时多态)与泛型(编译期多态)之间的胶水
- 核心思想: 将类型信息隐藏在内部虚函数后面，对外暴露类型无关的接口
- `std::function`是最实用的类型擦除，一般不要亲自手写
- `shared_ptr`的deleter是类型擦除的优势体现(deleter不是模板参数，不影响类型)

```cpp
// std::function擦除了callable的具体类型
std::function<int(int)> f;
f = [](int x) { return x * 2; };     // lambda
f = std::negate<int>{};                // 函数对象
f = some_free_function;                // 自由函数
// 调用方无需知道f内部存了什么
```

对比: `unique_ptr`的Deleter是模板类型参数 → 不同deleter产生不同类型
       `shared_ptr`的Deleter不是模板参数 → 类型擦除，所有`shared_ptr<T>`类型相同

Microsoft Proxy库(C++20): 非侵入式类型擦除，无需继承
```cpp
// 定义dispatch(相当于虚函数签名)
struct Draw : pro::dispatch<void(ostream&)> {
    template <class T>
    void operator()(const T& self, ostream& out) { self.Draw(out); }
};
struct DrawableFacade : pro::facade<Draw> {};

// 使用: 任何有Draw方法的类型都可以
void render(pro::proxy<DrawableFacade> p) {
    p.invoke<Draw>(cout);  // 无需继承
}
```

陷阱:
- 类型擦除必有效率降低(额外间接层、可能的堆分配)
- 手写type erasure复杂且易错，优先用std::function/std::any/proxy

---

## 7. RTTI(运行时类型信息)

信号: 需要在运行时确定对象的动态类型(typeid)或做安全向下转型(dynamic_cast)。

机制:
- RTTI仅对有虚函数的多态类有效
- `typeid(*ptr)` → 返回`const type_info&`(动态类型)
- `typeid(ptr)` → 返回指针本身的类型(不是指向对象的类型!)，必须解引用
- `dynamic_cast<Derived*>(base_ptr)` → 安全下转，失败返回nullptr
- `dynamic_cast<Derived&>(base_ref)` → 安全下转，失败抛`bad_cast`
- `type_info`不能作关联容器key → 用`type_index`包装
- gcc下`name()`返回混淆名，用`abi::__cxa_demangle()`获取可读名

```cpp
// UT中验证工厂产出类型(RTTI的正当使用场景)
auto product = factory.create("widget");
ASSERT_NE(dynamic_cast<Widget*>(product.get()), nullptr);

// type_index作为map的key
unordered_map<type_index, string> type_names;
type_names[type_index(typeid(int))] = "int";
```

使用原则:
- 业务代码中出现RTTI是坏味道 → 意味着里氏替换原则(LSP)未被遵守
- "A common code smell that frequently indicates an LSP violation is the presence of type checking code within a code block that is polymorphic."
- UT和基础库(反射、序列化)是RTTI的合理使用场景
- 非多态场景用typeid比dynamic_cast更轻量(编译期解析，常数时间)

陷阱:
- 对空指针`typeid(*p)` → 抛`bad_typeid`
- `typeid`忽略cv限定: `typeid(int) == typeid(const int)` 为true
- ISO不保证同一类型的不同typeid表达式指向同一type_info对象
  → 用`hash_code()`或`type_index`做比较，不要用指针比较

---

## 8. Mixin

信号: 需要给多个不相关的类添加相同的功能(如undo/redo、计数、序列化)，且功能可自由组合。

机制:
- 传统mixin: 多重继承混入功能类
- 模板化mixin(C++): 颠倒继承方向 — 父类作为模板参数传给子类
- 通过线性化继承链实现功能叠加: `Redoable<Undoable<Number>>`

```cpp
struct Number {
    typedef int value_type;
    int n;
    void set(int v) { n = v; }
    int get() const { return n; }
};

template <typename BASE, typename T = typename BASE::value_type>
struct Undoable : public BASE {
    T before;
    void set(T v) { before = BASE::get(); BASE::set(v); }
    void undo()   { BASE::set(before); }
};

template <typename BASE, typename T = typename BASE::value_type>
struct Redoable : public BASE {
    T after;
    void set(T v) { after = v; BASE::set(v); }
    void redo()   { BASE::set(after); }
};

using MyNumber = Redoable<Undoable<Number>>;
MyNumber n;
n.set(42); n.set(84);  // 84
n.undo();               // 42
n.redo();               // 84
```

陷阱:
- "The problem with mixins is... construction." 构造函数参数传递困难
- Mixin伴随多重继承 → 菱形继承问题
- 编译错误信息极不友好(深层模板嵌套)

---

## 9. static_cast vs dynamic_cast (向下转型)

信号: 需要将基类指针/引用转为派生类。

| 场景 | static_cast | dynamic_cast |
|------|-------------|--------------|
| 安全下转(基类指针确实指向派生类对象) | 返回正确地址 | 返回正确地址 |
| 不安全下转(基类指针指向基类对象) | 返回非空(危险! UB) | 返回nullptr(安全) |
| 运行时开销 | 零 | 虚表查找 |
| 要求 | 无 | 基类必须有虚函数 |

CRTP中static_cast的安全性:
- CRTP里`static_cast<Derived*>(this)`是安全的，因为this实际就是Derived对象
- 安全性由CRTP的继承约定保证: `class D : Base<D>`

决策: 确定类型 → static_cast (零开销); 不确定类型 → dynamic_cast (安全检查)

---

## 10. Delegates(C++委托)

信号: 需要存储和调用各种类型的回调(成员函数、lambda、静态函数)，且追求比std::function更好的性能。

机制:
- Delegate = 类型安全的函数指针包装器，支持成员函数绑定
- 核心技巧: IDelegate虚基类 + 不同实现(StaticDelegate, RawDelegate, LambdaDelegate)
- InlineAllocator: 32字节内联缓冲区避免小lambda的堆分配

```cpp
// 对比三种写法:
// 1. 裸函数指针 — 语法丑陋，不支持状态
int(Foo::*pf)(float) = &Foo::Bar;
(foo.*pf)(42);

// 2. std::function — 需要对象作为显式参数
std::function<int(Foo&, float)> f = &Foo::Bar;
f(foo, 42);

// 3. Delegate — 对象隐式绑定
auto d = Delegate<int, float>::CreateRaw(&foo, &Foo::Bar);
d.Execute(42);
```

MulticastDelegate: 支持多个监听者，仅限void返回值，执行期间安全处理解绑。

---

## 11. 多态实现方式选择决策树

核心问题: virtual vs CRTP vs Type Erasure vs Concepts vs Proxy，何时用哪个?

```
需要运行时多态(异构容器、插件)?
├── 是 → 需要非侵入式(不能改源码)?
│   ├── 是 → Type Erasure (std::function / Proxy库)
│   └── 否 → virtual继承
│       ├── 类型集合开放(将来会加新类型) → virtual继承
│       └── 类型集合封闭 → std::variant + std::visit
└── 否 → 编译期多态
    ├── C++20可用?
    │   ├── 是 → Concepts(最简洁)
    │   └── 否 → CRTP
    └── 需要混入功能(如运算符扩展)?
        └── CRTP Mixin
```

| 维度 | virtual | CRTP | Type Erasure | Concepts | Proxy |
|------|---------|------|-------------|----------|-------|
| 运行时/编译期 | 运行时 | 编译期 | 运行时 | 编译期 | 运行时 |
| 异构容器 | 可以 | 不可 | 可以 | 不可 | 可以 |
| 侵入性 | 需要继承 | 需要继承 | 不需要 | 不需要 | 不需要 |
| 开销 | vptr+间接调用 | 零 | 间接调用+可能堆分配 | 零 | 间接调用+SBO |
| 可读性 | 高 | 中(模板) | 低(实现复杂) | 高 | 中 |
| C++版本 | C++98 | C++98 | C++98(手写)/11(std::function) | C++20 | C++20 |

---

## 12. Abseil Tips精华(接口设计相关)

### #94 bool参数可读性

信号: 函数参数中有bool，调用处`f(true, false)`令人费解。

方案(按优先级递增):
1. 参数名注释: `f(/*remove_flags=*/false)` — clang-tidy可校验
2. 解释性变量: `const bool remove_flags = false; f(remove_flags);`
3. 枚举替代(最佳): `enum class RemoveFlags { kNo, kYes }; f(RemoveFlags::kNo);`

### #99 非成员接口礼仪

信号: 为类型定义`operator==`、`swap()`、hash等非成员扩展。

核心规则: 放在与被扩展类型相同的命名空间中，利用ADL。最佳做法是类内friend定义:
```cpp
class Key {
    friend bool operator<(const Key& a, const Key& b) { return a.s_ < b.s_; }
    friend void swap(Key& a, Key& b) { swap(a.s_, b.s_); }
};
```

禁忌: 不在测试文件中定义类型的operator(ODR风险)、不为protobuf生成类型添加扩展。

### #198 Tag Types

信号: 需要消歧模板类的多个构造函数重载，或在不可移动类型上使用就地构造。

```cpp
std::optional<Foo> o(std::in_place, 5, 10);       // 就地构造
std::variant<A, B> v{std::in_place_type<A>};       // 指定类型
```

### #218 FTADLE扩展点设计

信号: 库需要让用户为自定义类型扩展功能(如hash、序列化)。

模式: Friend Template ADL Extension
- 用项目前缀命名扩展点(如`AbslHashValue`)防止意外匹配
- 用户在自己的类中用friend模板函数实现扩展
- 实例: `AbslHashValue`、`AbslStringify`

### #234 参数传递方式

| 场景 | 方式 |
|------|------|
| 数值/枚举/sizeof<=16的trivial类型 | by value |
| 转移所有权的智能指针 | by value (move) |
| 字符串 | string_view |
| vector只读 | absl::Span<const T> |
| 可调用对象 | absl::FunctionRef |
| protobuf/大对象 | const T& |
| 可选参数 | const T* (nullable) |
| 需要自己副本的容器 | by value + std::move |

关键警告: 如果参数需要在函数调用结束后仍然存活(如被存储)，不要按引用传递 → use-after-free根源。

---

## 未能获取的链接

以下链接在WebFetch时未返回完整内容或被重定向，不影响知识完整性(核心内容已从其他来源覆盖):
- vishalchovatiya.com/crtp-c-examples (通过搜索和其他CRTP资料覆盖)
- CppCon 2014 Zach Laine "Pragmatic Type Erasure" (视频，无文字版)
- Sean Parent "Inheritance Is The Base Class of Evil" (视频，核心观点已融入Proxy库节)
