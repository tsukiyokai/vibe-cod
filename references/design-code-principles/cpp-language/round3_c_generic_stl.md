# Round 3: C底层技术、泛型编程与STL

## 来源

- $MAIN L9-L386 (C基础)、L1186-L1386 (模板元/函数式)、L1647-L1720 (STL/库)
- $SUB: 位操作奇技淫巧、alias-template、科里化讨论、排序和建堆的谓词用法、qsort学习笔记、lower_bound和upper_bound、stl优先队列、hwcstl优先队列、emplace_back vs push_back、cpp科目二、c科目一
- Abseil Tips: #1 #49 #86 #88 #93 #108 #112 #136 #140 #141 #144 #146 #147 #148 #165 #224 #227 #229 #232
- WebFetch: do-while(0)解释、container_of解释

---

## 1. C宏技术

### 1.1 RETURN_IF宏: do-while(0)与可变参数

信号: 需要在C/C++代码中写多语句宏、条件提前返回宏。

机制:
- `do { ... } while(0)` 让多语句宏在任何控制流上下文中表现为单条语句。不用`{}`的原因: 调用者加的分号会让`if-else`断裂。`do-while(0)`恰好消耗掉尾部分号。
- `(void)0, 0` 替代裸`0`: 抑制MSVC的C4127警告(conditional expression is constant)。
- `!!` 双重否定: 将非零值规范化为1, 确保布尔上下文的一致性(C中整型不自动截断为0/1)。
- 可变参数宏实现参数重载: 用`_ARG2(__VA_ARGS__, 2, 1, 0)`计数, 拼接展开到不同实现。

```c
#define RETURN_IF(C, R)          \
    do {                         \
        if (!!(C)) { return R; } \
    } while ((void)0, 0)

// 可变参数版: 支持1或2个参数
#define _ARG2(_0, _1, _2, ...) _2
#define NARG2(...) _ARG2(__VA_ARGS__, 2, 1, 0)
#define RETURN_IF(...) _ONE_OR_TWO_ARGS(NARG2(__VA_ARGS__), __VA_ARGS__)
```

陷阱:
- 宏体内的`break`/`continue`会被`do-while`捕获, 不会穿透到外层循环
- `#` stringification和`##` token pasting是宏的核心工具, 但二次展开需要间接层

### 1.2 container_of: 从成员地址反推宿主地址

信号: 侵入式数据结构(链表节点嵌入宿主结构体)、给出成员指针需要访问整体。

机制:
1. `offsetof(type, member)` 算出member在type内的字节偏移
2. 将member指针转为`char*`(因为偏移是字节单位), 减去偏移量, 得到宿主首地址
3. 转回宿主类型指针

```c
#define container_of(ptr, type, member) ({                    \
    const typeof(((type *)0)->member) *__mptr = (ptr);        \
    (type *)((char *)__mptr - offsetof(type, member)); })

// 典型用法: Linux内核链表
struct task_struct {
    struct list_head tasks;  // 嵌入的链表节点
    pid_t pid;
};
// 从tasks指针恢复task_struct指针
struct task_struct *t = container_of(node_ptr, struct task_struct, tasks);
```

陷阱:
- 必须转`char*`, 否则指针算术按所指类型大小为单位, 减出错误地址
- `typeof`是GCC扩展, 不是标准C; C++中可用`decltype`替代
- CANN中华为CSTL的`VOS_CONTAINER_OF`同理: `((type *)((char *)(ptr) - (uintptr_t)(&((type *)0)->member)))`

### 1.3 MIN宏: 双重求值陷阱

信号: 宏参数包含函数调用或有副作用的表达式。

```c
// 危险: foo(z)可能执行两次
#define min(X, Y) ((X) < (Y) ? (X) : (Y))
next = min(x + y, foo(z));  // foo(z)可能被调用两次!

// 安全: GCC扩展typeof + 中间变量
#define min(X, Y) ({           \
    typeof(X) x_ = (X);       \
    typeof(Y) y_ = (Y);       \
    (x_ < y_) ? x_ : y_; })
```

陷阱: 所有类函数宏都有此问题。C++中优先用`constexpr`函数或`std::min`。

### 1.4 混编宏: extern "C"

信号: C++代码调用C库函数、C头文件在C++中使用、链接时undefined reference。

机制: C++编译器对函数名做name mangling(加入参数类型信息), C不做。`extern "C"`告诉C++编译器按C的调用约定和符号命名规则处理。

```c
#ifdef __cplusplus
extern "C" {
#endif
void c_function(int x);
#ifdef __cplusplus
}
#endif
```

陷阱:
- 不能在`extern "C" {}`内部#include其他头文件 — 会把被包含头文件中的C++符号也强制为C链接
- 名称混淆是链接undefined reference的头号原因
- CANN场景: hccl的C API层必须用此模式, 内部C++实现通过extern "C"接口暴露

### 1.5 分支预测宏: likely/unlikely

信号: 性能关键路径上的条件分支, 且分支概率极度不均。

```c
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

if (unlikely(error_occurred)) {
    handle_error();
}
```

陷阱: 现代CPU分支预测器已经很强, likely/unlikely只在极端热路径上有可测量效果。滥用反而会误导编译器。

### 1.6 预定义宏: 隔离业务

信号: 需要在编译期区分平台、功能模块、调试/发布模式。

机制: 用`#ifdef FEATURE_X`在编译期隔离不同业务逻辑, 比运行时if判断更高效且不引入死代码。

---

## 2. C语法精要

### 2.1 双重否定 !!x

信号: C代码中需要规范化布尔值(确保结果为0或1)。

机制: C中任何非零值都是"真", 但gpio_get_value()可能返回任意正整数。`!!x`将其规范化为0或1。C++中`static_cast<bool>`更规范。

### 2.2 Designated Initializers

信号: 数组/结构体初始化需要按名/按索引赋值, 表驱动设计。

```c
// 一维数组: 枚举值到字符串的映射(表驱动)
static const char *info_postfix[] = {
    [INFO_RAW] = "raw",
    [INFO_SCALE] = "scale",
};

// 结构体: 只初始化部分成员, 其余默认为0
struct cfg cfg = { .timeout = 30, .retries = 3 };
```

要点:
- C99/C++20可用
- 静态数组中未指定的位置天然为0
- 是实现表驱动设计的基础工具, 用枚举值做索引可以大幅提高可读性

### 2.3 复合字面量

信号: 需要临时创建结构体/数组值作为函数参数或赋值。

```c
// 复合字面量是左值, 可以取地址
int *p = (int[]){1, 2, 3};

// 文件作用域: 静态存储期; 块作用域: 自动存储期
```

### 2.4 赋值规则

- 同构兼容类型的结构体之间可以直接赋值
- 数组之间不能直接赋值(数组是二等公民), 但包在结构体里可以
- 函数是一等公民, 数组是二等公民

### 2.5 常量在左

信号: `if (x == CONST)` 有写成 `if (x = CONST)` 的typo风险。

`if (CONST == x)` 写反为 `if (CONST = x)` 时编译器会报错, 因为不能给常量赋值。现代编译器的`-Wparentheses`也能检测, 但常量在左仍是一些代码规范的要求。

---

## 3. 位操作

### 3.1 核心位操作技巧

信号: 底层编程、标志位管理、性能优化(避免分支/除法)。

| 技巧 | 代码 | 用途 |
|------|------|------|
| 判奇偶 | `n & 1` | 替代 `n % 2` |
| 判2的幂 | `n && !(n & (n-1))` | 消去最右边的1 |
| 找唯一数 | 全部XOR | O(1)空间, 利用`n^n=0, n^0=n` |
| 位设置 | `byte \|= (1<<n)` | set nth bit |
| 位清除 | `byte &= ~(1<<n)` | clear nth bit |
| 位翻转 | `byte ^= (1<<n)` | flip nth bit |
| 位检查 | `byte & (1<<n)` | check nth bit |

```c
// 十六进制转换: 用字符串查表代替分支
"0123456789ABCDEF"[(n >> i*4) & 0xF]
```

### 3.2 无分支位操作

信号: 高性能计算中避免分支预测失败的性能损失。

```c
// 判断两数符号相反
(x ^ y) < 0

// 无分支求绝对值
int mask = v >> (sizeof(int) * CHAR_BIT - 1);
r = (v + mask) ^ mask;

// 无分支min/max
r = y ^ ((x ^ y) & -(x < y));  // min(x, y)
r = x ^ ((x ^ y) & -(x < y));  // max(x, y)
```

### 3.3 加法的位运算分解

```c
x + y = (x ^ y) + ((x & y) << 1)
// XOR是不带进位的加法, AND<<1是进位部分
```

布尔代数恒等式在实际代码中的应用: `~(x | y) = ~x & ~y` (De Morgan定律) 可用于化简复杂位操作表达式。

陷阱:
- 位操作优先级低于比较运算符: `n & (n-1) == 0` 实际解析为 `n & ((n-1) == 0)`, 必须加括号
- 有符号整数的右移是实现定义的(算术移位vs逻辑移位)
- 混合有符号/无符号位操作时容易出现符号扩展问题

---

## 4. 泛型编程

### 4.1 模板基础认知

信号: 需要理解模板的编译模型、何时用显式实例化。

核心认知:
- "泛化以扩大外延, 特化以增加内涵" — 模板设计的指导原则
- 模板复用源代码, 继承/组合复用对象代码 — 模板的复用能力从根源上更强
- 模板类没有实例化就没有定义只有声明

```cpp
// 隐式实例化: 使用时自动触发
std::vector<int> v;

// 显式实例化: 手动触发, 控制编译时间
template class std::vector<int>;

// 好的实践: 用using别名简化复杂模板类型
using ScoreMap = std::unordered_map<std::string, std::vector<int>>;
```

### 4.2 模板名字查找: 两阶段与依赖名

信号: 模板基类的成员在派生类模板中直接调用时编译报错"no arguments that depend on a template parameter"。

机制: 模板编译分两阶段:
1. 定义时: 检查不依赖模板参数的名字(non-dependent names)
2. 实例化时: 检查依赖模板参数的名字(dependent names)

基类是模板时, 基类成员是dependent name, 第一阶段找不到。

```cpp
template <typename T>
class Base { public: void Method(); };

template <typename T>
class Derived : public Base<T> {
    void Func() {
        Method();          // 错! 第一阶段找不到
        this->Method();    // 对: 通过this使其成为dependent name
        Base<T>::Method(); // 对: 显式限定
    }
};
```

### 4.3 SFINAE与模板形参的编译期语义

信号: 看到"没被使用"的模板参数; 需要根据类型特征选择不同实现。

核心: 模板形参即使不在函数体中使用, 也可能承担编译期语义:

| 场景 | 实际价值 |
|------|---------|
| `enable_if_t<...>` 默认参数 | 控制是否参与重载集合 |
| CRTP占位: `template<class Derived, class Tag = void>` | 扩展点预留 |
| 非类型参数`template<int ID>` | 不同ID生成不同符号, 用于调试/度量 |
| `template<class T, class = void>` | ABI兼容占坑, 后期可扩展 |

```cpp
// SFINAE: enable_if控制重载
template <class T>
auto print(T v) -> std::enable_if_t<std::is_integral_v<T>> {
    std::cout << v << '\n';
}

// C++20 concepts更直观
template <std::integral T>
void print(T v) { std::cout << v << '\n'; }
```

### 4.4 Ranked Overloads: SFINAE的优先级排序 (Abseil #229)

信号: 多个模板重载同等匹配导致歧义, 需要显式优先级。

```cpp
struct Rank0 {};
struct Rank1 : Rank0 {};
struct Rank2 : Rank1 {};  // 编译器优先选择更派生的类型

template <typename T>
auto SizeImpl(Rank2, const T& x) -> decltype(x.length()) { return x.length(); }

template <typename T>
auto SizeImpl(Rank1, const T& x) -> decltype(x.size()) { return x.size(); }

template <typename T>
size_t SizeImpl(Rank0, const T&) { return 1; }  // fallback

template <typename T>
size_t Size(const T& t) { return SizeImpl(Rank2{}, t); }
```

陷阱: 重载签名中用具体类型(如`string_view`)而非模板参数时, 隐式转换会破坏rank优先级。

### 4.5 alias-template: using vs typedef

信号: 需要给模板类型起别名, 且别名本身也需要是模板。

```cpp
// typedef无法模板化
// using可以
template <typename T>
using Vec = std::vector<T>;

Vec<int> v;  // 等价于 std::vector<int> v;
```

### 4.6 auto推导规则

信号: 使用auto声明变量时的类型推导行为。

核心规则(来自cpp科目二):
- 声明为非指针/非引用时: auto = 原表达式抛弃引用 + 抛弃cv
- 声明为指针/引用时: auto = 原表达式抛弃引用 + 保留cv

```cpp
const int& cr = 42;
auto a = cr;         // int (抛弃const和引用)
auto& b = cr;        // const int& (保留const, 因为是引用声明)
const auto& c = cr;  // const int& (显式加const)
auto* d = &cr;       // const int* (保留const, 因为是指针声明)
```

Abseil #232补充 — 何时用auto:
- 该用: range-for遍历map(避免隐式拷贝)、迭代器声明、工厂函数右侧已有类型
- 不该用: 当auto会隐藏关键语义信息时(是引用还是拷贝? 是否const?)

```cpp
// 危险: pair类型写错导致每次迭代都拷贝
for (const std::pair<std::string, V>& p : map) { ... }
//                   ^^^ 应该是const std::string

// 安全: auto + structured bindings
for (const auto& [name, value] : map) { ... }
```

### 4.7 ADL (Argument-Dependent Lookup) (Abseil #49)

信号: 非限定函数调用的行为出乎意料; 重构时调用目标静默改变。

机制: 调用`func(a)`时, 编译器除了在词法作用域查找, 还会在a的类型所属命名空间中查找。

```cpp
namespace aspace {
    struct A {};
    void func(const A&);
}
void func(int);
void test() {
    aspace::A a;
    func(a);  // 调用aspace::func, 不是全局的func!
}
```

陷阱:
- 迭代器的命名空间是平台相关的, 对STL算法务必写全限定名`std::count()`
- operator重载必须定义在操作数类型的命名空间里, 否则可能被劫持
- `using`别名会被展开后再确定关联命名空间

---

## 5. 函数式风格: 柯里化决策

信号: 定义多参数函数时, 如何选择参数传递方式。

三种定义多参函数的风格:
1. 语法层面的多参: `int plus(int a, int b)` — 简单直接, 但高阶抽象时传参困难
2. 组合型参数(元组): `int plus(std::pair<int,int> p)` — 高阶函数友好
3. 柯里化: `auto plus(int a) { return [a](int b) { return a+b; }; }` — 便于部分应用

何时柯里化: 先指定某些参数就有明确意义时(如`all(judge)`, 固定judge后得到`judge_list`)
何时不柯里化: 参数必须同时给出才有意义时(如坐标x,y — 给出单独的x没有语义)

```cpp
// 部分应用的实际例子
auto all_positive = [](auto&& range) {
    return std::ranges::all_of(range, [](int x){ return x > 0; });
};
```

C++中的启示:
- Abseil #108: 避免`std::bind`, 用lambda代替 — lambda更清晰, 支持move-only类型
- `std::bind_front`(C++20)适合简单的部分应用
- 柯里化倾向于让人写出过多细节的代码, 缺少必要的中间抽象步骤

---

## 6. STL实战

### 6.1 排序和建堆的谓词统一理论

信号: 每次写比较函数都要纠结greater/less、升序降序。

统一记忆法:
- C++ `bool cmp(a, b)`: a和b在原数组中的位置就是`a b`(从左到右)
- C `int cmp(a, b)`: a和b在原数组中的位置是`b a`(相反!)
- 交换规则统一: 返回真(正数)时不动, 返回假(负数)时交换

```
C++: cmp(a,b) -> 原始位置 a b
  return a < b -> true  -> 不交换 -> a在前b在后 -> 升序
  return a > b -> true  -> 不交换 -> a在前(大)b在后(小) -> 降序

C:   cmp(a,b) -> 原始位置 b a
  return a - b > 0 (a > b) -> 不交换 -> b在前(小)a在后(大) -> 升序
```

优先队列的谓词:
- STL `priority_queue`默认大顶堆(`less<T>`)
- 想要小顶堆: `priority_queue<int, vector<int>, greater<int>>`

### 6.2 qsort: Sedgewick四项优化

信号: 理解GNU实现的qsort优化策略。

```
GNU qsort = quicksort(大分区) + insertion sort(小分区, <=4个元素)
```

四项优化:
1. 非递归: 用显式栈替代递归调用, 栈空间仅需`CHAR_BIT * sizeof(size_t)`字节
2. 三数取中: 选lo/mid/hi的中位数做pivot, 降低最坏情况概率
3. 小分区用插入排序: 当分区<=MAX_THRESH(4)时, 插入排序更快(数据几乎有序)
4. 大分区先压栈: 先处理小分区, 保证栈深度不超过O(log n)

qsort比较函数惯用法:
```c
int cmp(const void *a, const void *b) {
    int x = *(const int *)a, y = *(const int *)b;
    return (x > y) - (x < y);  // 安全: 不会溢出
    // 不要用 return x - y;     // 危险: INT_MIN - 1 溢出
}
```

### 6.3 lower_bound vs upper_bound

信号: 二分查找时不确定用哪个。

```
lower_bound: 返回第一个 >= x 的位置
upper_bound: 返回第一个 >  x 的位置

[lower_bound, upper_bound) = 所有等于x的元素范围
upper_bound - lower_bound = 等于x的元素个数
```

实现差异仅一个符号:
```c
// lower_bound: x <= a[m] 时移右界
if (x <= a[m]) h = m; else l = m + 1;

// upper_bound: x >= a[m] 时移左界
if (x >= a[m]) l = m + 1; else h = m;
```

### 6.4 emplace_back vs push_back (Abseil #112)

信号: 选择emplace还是push。

Abseil官方建议: 默认用`push_back`, 除非benchmark证明emplace有显著收益。

原因:
- `push_back`类型不匹配时编译报错; `emplace_back`因为直接转发参数, 可能把错误参数"成功"编译
- 例: `vec<vector<int>>.emplace_back(1<<20)` 默默创建百万元素vector, 而`push_back(1<<20)`编译报错

```cpp
// 默认: push_back
vec.push_back(MyObj{arg1, arg2});

// 仅当存在隐式转换且性能关键时: emplace_back
vec.emplace_back("string_literal");  // 避免const char*->string的临时对象
```

### 6.5 容器使用要点

vector:
- `push_back`可能触发重新分配, 导致所有迭代器/指针/引用失效
- 不要用`at()` (Abseil #224): 已知索引合法时多余, 索引可能非法时异常不是好的错误处理。用`operator[]`配合显式边界检查
- `v.size() - 1`在空容器上回绕为`~0ULL` (Abseil #227): 把`i < size - 1`改写为`i + 1 < size`

```cpp
// 危险: 空容器时size()-1回绕
for (size_t i = 0; i < v.size() - 1; ++i) { ... }
// 安全: 减法改加法
for (size_t i = 0; i + 1 < v.size(); ++i) { ... }
```

unordered containers (Abseil #136):
- 新代码优先用`absl::flat_hash_map/set` — 值直接存在主数组中, 缓存友好
- 需要指针稳定性: `flat_hash_map<K, unique_ptr<V>>` 优于 `node_hash_map`
- 陷阱: `map["a"] = map["b"]` 在flat hash map上可能触发rehash导致访问失效内存

heterogeneous lookup (Abseil #144):
- 透明比较器让`map.find(string_view)`无需构造临时string
- 需要显式opt-in(`is_transparent`), 因为不同类型间的语义关系可能不一致

### 6.6 华为CSTL优先队列

信号: CANN生态中使用C语言的优先队列实现。

华为CSTL的`VosPriQue`基于libpqueue(Apache HTTP Server使用的堆实现):
- 核心操作: `bubble_up`(上浮)和`percolate_down`(下沉)
- 用函数指针实现泛型比较: `VosPriQueCmpFunc`
- 内嵌`VOS_CONTAINER_OF`宏: 与Linux kernel的`container_of`同源
- 注意`VOS_IntCmpFunc`直接做减法`data1 - data2`, 有溢出风险(同qsort的cmp陷阱)

---

## 7. 初始化与类型安全 (Abseil Tips精华)

### 7.1 初始化三板斧: =, (), {} (Abseil #88)

| 场景 | 语法 | 例子 |
|------|------|------|
| 直接赋值/字面量/容器列表 | `=` | `int x = 2;` `vector<int> v = {1,2,3};` |
| 涉及计算/构造逻辑 | `()` | `vector<double> pies(50, 3.14);` |
| 以上都编不过时 | `{}` | 成员初始化列表 `array_{a,b,c}` |

陷阱:
- `vector<string> strings{2}`: 创建2个空string
- `vector<int> ints{2}`: 创建包含元素2的vector
- 同样语法, 因initializer_list优先级不同而语义完全不同
- 不要把auto和大括号混用: `auto x{1}` vs `auto y = {1}` 类型不同

### 7.2 Default vs Value初始化 (Abseil #146)

```cpp
struct Foo {
    Foo() {}    // user-provided构造函数
    int v;      // Foo f; -> v未初始化, 读取是UB
};

struct Bar {
    Bar() = default;  // user-declared but not user-provided
    int v;
    // Bar b = {}; -> v零初始化
    // Bar b;      -> v未初始化!
};
```

安全做法: 用default member initializer
```cpp
struct Safe {
    int v = 0;  // 无论如何构造都会初始化
};
```

### 7.3 常量的安全惯用法 (Abseil #140)

```cpp
// C++17头文件中: inline确保全程序单一实例
inline constexpr int kMyNumber = 42;
inline constexpr absl::string_view kMyString = "Hello";

// 危险: 无inline的constexpr在头文件中每个翻译单元各一份拷贝
// 地址不同, 指针比较失败
```

### 7.4 隐式bool转换 (Abseil #141)

```cpp
// optional<bool>(false) 在if中为true!
std::optional<bool> b = false;
if (b) { ... }             // 执行! 因为optional有值
if (b.has_value()) { ... } // 意图明确: 检查是否有值
if (*b) { ... }            // 意图明确: 检查值本身
```

### 7.5 enum class (Abseil #86)

```cpp
// 传统enum: 枚举值污染命名空间, 可隐式转int
enum Color { kRed, kGreen };
int i = kRed;  // 编译通过

// enum class: 强类型, 不污染, 不隐式转换
enum class Color { kRed, kGreen };
Color c = Color::kRed;
int i = Color::kRed;  // 编译错误!
```

### 7.6 穷举switch (Abseil #147)

不写default的穷举switch, 仅在枚举值集合稳定时使用。

```cpp
std::string ToString(Status s) {
    switch (s) {
        case Status::kOk:    return "ok";
        case Status::kError: return "error";
        // 无default -- 新增枚举值时编译器会警告
    }
    return "unknown";  // switch之后兜底, 处理非法值
}
```

### 7.7 if/switch with initializers (Abseil #165)

```cpp
// C++17: 变量作用域严格限制在if块内
if (auto it = m.find("key"); it != m.end()) {
    return it->second;
}
// it在这里不可见, 不会污染外部作用域

// 与structured bindings结合
if (auto [iter, inserted] = m.try_emplace(key, val); inserted) {
    use(iter->second);
}
```

---

## 8. 接口设计补充 (Abseil Tips)

### 8.1 string_view与Span (Abseil #1, #93)

- `string_view`: 只读字符串参数的首选类型, 指针+长度, 按值传递, 零拷贝
- `absl::Span<const T>`: 函数参数用Span替代`const vector<T>&`, 可接受vector/数组切片/initializer_list
- 两者共同陷阱: 不拥有数据, 必须确保底层数据比view活得更久; 按值传而非按引用传

### 8.2 Overload Sets (Abseil #148)

核心规则: 重载集中所有函数必须具有相同语义。判断标准: 读者在调用点不需要知道具体调用了哪个重载。

### 8.3 避免std::bind (Abseil #108)

```cpp
// 差: std::bind
auto f = std::bind(&Foo::OnDone, this, std::placeholders::_1);

// 好: lambda
auto f = [this](auto&& arg) { OnDone(arg); };

// 好: C++20 bind_front (简单partial application)
auto f = std::bind_front(&Foo::OnDone, this);
```

bind的问题: 静默丢参数、不支持move-only类型、嵌套bind行为反直觉。

---

## 9. 考试要点精选

### 9.1 noexcept传播

虚函数有`noexcept`, 则覆盖函数必须也写`noexcept`。
noexcept不能和typedef组合使用。

### 9.2 new bool[]

`new bool[3]` 不保证初始化为0。基本数据类型只有全局变量才会0初始化, 堆上分配的不保证。
安全做法: `new bool[3]()` 或用`vector<bool>`。

### 9.3 static变量递归初始化

```cpp
int test(int i) {
    if (i <= 0) return 0;
    static int y = test(i - 1);  // 递归触发同一static的初始化
    return y + 1;
}
// 结果: std::__gnu_cxx::recursive_init_error
```

C++11起static局部变量初始化是线程安全的(只发生一次), 但递归初始化同一static变量是UB。

### 9.4 虚函数与内联

- 通过对象调用: 编译期已知确切类型, 可以内联
- 通过基类指针/引用调用: 运行时多态, 不能内联

### 9.5 头文件包含顺序

1. 对应的.h文件
2. STL头文件
3. 系统库头文件
4. 项目内其他头文件

### 9.6 extern "C"中不能#include

不能在`extern "C" {}`内部#include其他头文件, 会把被包含头文件中的C++符号强制为C链接。

---

## 外部链接访问记录

成功WebFetch:
- Abseil Tips #1, #49, #86, #88, #93, #108, #112, #136, #140, #141, #144, #146, #147, #148, #165, #224, #227, #229, #232
- do-while(0)解释 (通过SEI CERT和Codidact替代源)
- container_of解释 (通过EmbeTronicX和radek.io替代源)

未单独WebFetch (内容已从源材料充分提取):
- bithacks.html (源材料中已有核心技巧)
- gcc designated inits (源材料中已覆盖)
- gcc variadic macros (源材料中已覆盖)
