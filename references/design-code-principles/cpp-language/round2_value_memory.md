# Round 2: 值语义、对象模型与内存管理

## 来源

- $MAIN L388-L936 (C++基础: 值类别、类型转换、指针、数组、引用、移动语义、函数、对齐/布局/内存/对象模型、联合体)
- $SUB: 左右值探秘by苏、new in cpp、emplace_back vs push_back、不透明指针、前向声明、Pimpl、数组引用、多维数组的堆内存申请方式、const_cast
- WebFetch: 0xghost std::move deep dive、PVS-Studio move misconceptions (V828/V833/V839)、ESR The Lost Art of Structure Packing、Qt D-Pointer Wiki
- Abseil Tips: #11/#24/#55/#77/#101/#107/#112/#117/#120/#126/#131/#134/#166/#176/#180/#187/#188

---

## 1. 值类别

信号: 理解表达式是lvalue还是rvalue，决定了能否取地址、能否绑定到哪种引用、是否触发移动语义。

机制:
- 每个C++表达式由两个维度分类: 有没有身份(identity)? 资源能否被移走?
- lvalue: 有地址且可访问(变量、通过引用返回的函数)
- xvalue(expiring value): 有地址但被标记为即将过期(std::move产生的就是xvalue)
- prvalue(pure rvalue): 没有地址(字面量42、临时对象`string("hello")`)
- 左值是判断出来的，右值是排除出来的

```
        glvalue              rvalue
       /      \            /      \
   lvalue    xvalue    xvalue    prvalue

glvalue = lvalue + xvalue (有身份)
rvalue  = xvalue + prvalue (可移动)
xvalue  = 两者交集 (有身份且可移动)
```

关键绕口:
- 右值引用变量本身是左值: `int&& r = 42;` 中r是左值(有名字、可取地址)
- const左值引用可绑定右值: `const int& cr = 42;` 合法，避免不必要的拷贝
- C中右值永远没有cv限定; C++中类类型的右值可以有cv限定(内置类型不行)

数组退化三个例外(C90):
1. `sizeof(arr)` — 返回整个数组大小，不退化为指针
2. `&arr` — 产生指向整个数组的指针，不退化
3. 字符串字面量初始化`char`数组 — 不退化

```cpp
int a[2] = {1, 2};
// a+1:  跳一个int的步长 → a+4字节
// &a+1: 跳整个数组的步长 → a+8字节
// a和&a值相同，但类型不同!
```

陷阱:
- 返回引用的函数是左值，可以作为赋值目标: `map[10] = 5.6;` — operator[]返回引用
- 可修改左值排除了: const修饰、数组类型、不完整类型、含const成员的聚合类型
- C语言的"左值"原始含义是"可放在赋值号左边的值"，const出现后细化为"可修改左值"

---

## 2. 移动语义与完美转发

信号: 需要避免不必要的深拷贝，或需要将参数原样传递给下一层函数。

### 2.1 std::move的本质

机制: std::move不移动任何东西，它只是一个`static_cast`。

```cpp
// std::move的标准库实现
template<typename T>
constexpr std::remove_reference_t<T>&&
move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
// 剥离引用限定符 + 添加&& → 将lvalue转为xvalue
// 真正的资源转移发生在移动构造函数/移动赋值运算符被调用时
```

### 2.2 五大误区(来自0xghost + PVS-Studio + pikoTutorial)

误区1 — return中使用std::move阻止NRVO:
```cpp
// 错误: 阻止了命名返回值优化
std::string createString() {
    std::string result = "data";
    return std::move(result);    // 强制执行一次move
}

// 正确: 让编译器做NRVO，零开销
std::string createString() {
    std::string result = "data";
    return result;               // 直接在调用者内存中构造
}
```
PVS-Studio V828专门检测此反优化模式。C++ Core Guidelines F.48: Do not return std::move(local)。

误区2 — 对const对象std::move静默退化为拷贝:
```cpp
const std::vector<int> data = getData();
consume(std::move(data));        // 编译通过，但实际执行拷贝!
// const T&& 无法绑定到 T&&，编译器回退到 const T& (拷贝构造)
```
PVS-Studio V833检测此问题。危险在于编译器不报错，性能悄悄退化。

误区3 — move后继续使用源对象:
```cpp
std::string name = "Alice";
std::string moved = std::move(name);
std::cout << name;               // 危险: name处于"valid but unspecified"状态
// 安全操作: 销毁它 或 赋予新值
```

误区4 — std::move无法让不可移动类型变得可移动:
```cpp
class NonMovable {
    NonMovable(NonMovable&&) = delete;
};
NonMovable obj2(std::move(obj1));  // 编译错误!
```

误区5 — 基本类型move后值不变:
```cpp
int x = 42;
int y = std::move(x);
// x仍然是42 — 基本类型没有移动构造函数，move退化为拷贝
```

额外 — 函数返回const值类型阻止移动语义:
```cpp
// 错误: const返回类型阻止move
const std::vector<Object> GetAll() { ... }

// 正确: 移除const允许move赋值
std::vector<Object> GetAll() { ... }
```
PVS-Studio V839检测，对应C++ Core Guidelines F.49。

### 2.3 noexcept的关键性

移动构造/赋值必须标记`noexcept`。

原因: std::vector扩容时需要搬元素到新内存。若move构造可能抛异常，vector无法保证强异常安全(搬到一半抛异常，原vector已部分破坏)，因此回退为逐个拷贝。

```cpp
// 移动构造: 必须noexcept
Resource(Resource&& other) noexcept
    : data(std::exchange(other.data, nullptr)),
      size(std::exchange(other.size, 0)) {}
```

性能基准(来自0xghost):

| 操作 | 耗时 | 相对性能 |
|------|------|---------|
| 深拷贝 | 7.82 ms | 基准 |
| 正确移动 | 1.08 ms | 快约7倍 |
| 对const移动 | 7.50 ms | 退化为拷贝 |
| 有noexcept的扩容 | 1.63 ms | 基准 |
| 无noexcept的扩容 | 16.42 ms | 慢10倍 |

### 2.4 std::move vs std::forward

- std::move: 无条件转为rvalue。语义="我不再需要这个对象"
- std::forward: 有条件转换，保留原始值类别。仅用于模板中的转发引用(universal reference)

### 2.5 Rule of Five完整实现

```cpp
class Resource {
    int* data;
    size_t size;
public:
    ~Resource() { delete[] data; }

    // 拷贝构造: 深拷贝
    Resource(const Resource& other)
        : data(new int[other.size]), size(other.size) {
        std::copy(other.data, other.data + size, data);
    }

    // 拷贝赋值: 深拷贝 + 强异常保证(先new再delete)
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            int* new_data = new int[other.size];
            std::copy(other.data, other.data + other.size, new_data);
            delete[] data;
            data = new_data;
            size = other.size;
        }
        return *this;
    }

    // 移动构造: noexcept + std::exchange
    Resource(Resource&& other) noexcept
        : data(std::exchange(other.data, nullptr)),
          size(std::exchange(other.size, 0)) {}

    // 移动赋值: noexcept + std::exchange
    Resource& operator=(Resource&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = std::exchange(other.data, nullptr);
            size = std::exchange(other.size, 0);
        }
        return *this;
    }
};
```

陷阱:
- Small String Optimization(SSO): 小字符串直接存在对象内部，移动时仍需复制字节，并非零开销
- 虚函数有noexcept则覆盖函数必须也写noexcept

---

## 3. 对象模型与内存布局

信号: 需要理解sizeof结果、优化结构体大小、跨进程/跨机器通信的数据序列化。

### 3.1 对齐与填充

机制: 内存对齐是空间换时间。未对齐的数据在某些平台上会引发硬件异常。

核心规则(来自ESR "The Lost Art of Structure Packing"):
- 自对齐(self-alignment): 每种类型按自身大小对齐。`char`=1字节对齐, `short`=2, `int`=4, `long`/指针=8(64位)
- 结构体对齐 = 成员中最大对齐值
- 结构体末尾padding: 确保数组中下一个元素正确对齐(stride address)
- padding只出现在两个地方: (1)严格对齐的大类型跟在小类型后面 (2)结构体末尾

```cpp
// 原始布局: 24字节(8+1+7padding+8)
struct foo10 {
    char c;          // 1 byte
    char pad1[7];    // 7 bytes padding
    struct foo10* p; // 8 bytes
    short x;         // 2 bytes
    char pad2[6];    // 6 bytes padding
};  // 总计24字节

// 重排后: 16字节(8+2+1+5padding)
struct foo11 {
    struct foo11* p; // 8 bytes
    short x;         // 2 bytes
    char c;          // 1 byte
    char pad[5];     // 5 bytes padding
};  // 总计16字节
```

重排优化: 按对齐值递减排列成员(指针 → long → int → short → char)，可有效消除内部padding。递增排列也能达到同样效果。

### 3.2 sizeof规则速查表

| 场景 | sizeof | 说明 |
|------|--------|------|
| 空叶子类 | 1 | 每个对象必须有唯一地址 |
| 空基类(EBO) | 0 | 空基类优化，不占派生类空间 |
| 只含char的类 | 1 | 与空叶子类相同 |
| 有成员函数(非虚) | 不变 | 成员函数不占对象空间 |
| 有虚函数(不管几个) | +1个vptr | 通常+4/8字节(32/64位) |
| 静态成员 | 不算 | 跟类走，不跟对象走 |

- 非静态成员变量跟对象走(对象内)
- 静态成员变量跟类走(对象外)
- 虚函数表跟类走，vptr跟对象走

### 3.3 EXP42-C: 不要比较padding数据

由于padding位的值未定义，使用`memcmp`比较结构体是不安全的。应该逐字段比较，除非确认padding已被清零且运行速度极其重要。

### 3.4 this指针调整

多重继承时，基类子对象的地址可能与派生类对象地址不同。编译器在虚函数调用时自动调整this指针，使其指向正确的基类子对象。

### 3.5 Itanium ABI

C++ ABI规范精确定义了C++对象在机器码中如何实现: name mangling、虚函数表布局、异常处理等。理解ABI对跨编译器/跨版本的二进制兼容性至关重要。

Visual Studio有StructLayout扩展可可视化C++结构体内存布局。

陷阱:
- C++类有虚函数时，结构体首地址可能不等于第一个成员地址(因为vptr)
- `#pragma pack`强制打包会生成更慢的代码(非对齐访问)，仅在匹配硬件/协议布局时使用
- 多线程场景: 读写分离到不同cache line以避免cache line bouncing(false sharing)
- 64位x86 cache line = 64字节，热点数据应尽量fit进一个cache line

---

## 4. 联合体

信号: 需要同一段内存按不同方式解释，或在资源受限平台上节省内存。

机制: 联合体的所有数据成员共享同一内存，大小 = 最大成员的大小。

三种经典用法(ugly and hacky):

用法1 — 逐字节序列化:
```c
union employeeData {
    struct eData sEmployee;
    char cEmployeeSerialized[64];
} employee;
// 写入结构体字段，通过char数组逐字节发送串口
uartTxBuffer(&employee.cEmployeeSerialized);
```

用法2 — 通用缓冲区(节省内存):
```c
union multiPurposeBuffer {
    float modulatorOutput[256];
    int interleaverOutput[256];
    char scramblerOutput[1024];
} multiBuffer;  // 只占1024字节而非2048
```

用法3 — type punning(以不同方式解释同一内存):
```c
typedef union {
    struct {
        unsigned short int booster : 1;
        unsigned short int retro   : 1;
        // ... 更多位域
    } u;
    unsigned short int TotalFlightStatus;
} NASAFlightStatus;  // 可按位操作，也可整体读写
```

Tagged union(类型安全):
```cpp
// C风格: enum tag + 匿名union
struct S {
    enum { CHAR, INT, DOUBLE } tag;
    union { char c; int i; double d; };
};
void print_s(const S& s) {
    switch (s.tag) {
        case S::CHAR:   cout << s.c; break;
        case S::INT:    cout << s.i; break;
        case S::DOUBLE: cout << s.d; break;
    }
}

// C++17替代: std::variant
std::variant<char, int, double> s = 'a';
std::visit([](auto x) { cout << x << '\n'; }, s);
```

类型安全的C语言继承(利用union):
```c
// 利用union让base指针可安全转为不同子类型
// 参考: http://www.deleveld.dds.nl/inherit.htm
```

陷阱:
- 读非最后写入的成员是reinterpret，可能是trap representation(C标准6.2.6)
- 字节序问题: union type punning在大端/小端机器上行为不同
- C++17起，优先使用`std::variant`替代手写tagged union
- union中有非平凡成员(如std::string)时，构造/析构变得复杂

---

## 5. new/placement new

信号: 需要动态分配对象，或需要分离内存分配与对象构造(内存池、allocator)。

### 5.1 new的4种语法

```cpp
::new (type) initializer            // 1. 简单形式
::new new-type initializer          // 2. 无括号(注意运算符优先级)
::new (placement-params) (type) initializer  // 3. placement new
::new (placement-params) new-type initializer // 4. placement new无括号
```

初始化规则:
- 无初始化器 → 默认初始化
- 括号参数列表 → 直接初始化
- 花括号参数列表 → 列表初始化

### 5.2 placement new

核心: 在已分配的内存上构造对象，分离分配与构造。

```cpp
char memory[sizeof(Fred)];
void* place = memory;
Fred* f = new(place) Fred();    // 在指定内存位置构造
// f和place指向同一地址

// 必须显式调用析构函数，不能delete!
f->~Fred();
```

placement new在内存池中的核心作用:
```cpp
const int size = 10;
char buf[size * sizeof(A)];          // 预分配内存

for (size_t i = 0; i < size; i++) {
    new (buf + i * sizeof(A)) A(i);  // 原地构造
}

A* arr = (A*)buf;
for (size_t i = 0; i < size; i++) {
    arr[i].~A();                     // 显式析构
}
// buf是栈内存，自动回收
```

为什么预分配用char数组:
1. 方便控制大小(sizeof计算)
2. 不会触发自定义类型的构造函数(分配与构造分离的初衷)

### 5.3 内存泄漏三种场景

```cpp
// 场景1: 指针被覆盖
int* p = new int(7);
p = nullptr;             // 泄漏: 原始地址丢失

// 场景2: 指针越域
void f() {
    int* p = new int(7);
}                         // 泄漏: p超出作用域

// 场景3: 异常跳过delete
void f() {
    int* p = new int(7);
    g();                  // 可能抛异常
    delete p;             // 异常时不执行
}                         // 解法: 用智能指针
```

陷阱:
- placement new的对齐责任完全由使用者承担 — 编译器/运行时不检查
- placement new构造的对象不能delete，必须显式调析构函数
- placement new本质上是new操作符的重载
- `new (this) Resource()` — 在析构函数中重置对象的技巧(罕见但存在)

---

## 6. emplace_back vs push_back

信号: 向容器添加元素时，选择push_back还是emplace_back。

机制:
- emplace_back: 直接将参数转发给构造函数，在容器中原地构造(底层是placement new)，0拷贝
- push_back: 可能先构造临时对象再移动到容器中

```cpp
std::vector<std::string> words;
words.push_back("hello");      // 字符串字面量 → 构造临时string → 移入容器
words.emplace_back("hello");   // 直接在容器内存中构造string，省一次移动

std::list<President> presidents;
presidents.push_back(President("Obama", 2009));  // 构造临时 + 移动
presidents.emplace_back("Obama", 2009);          // 直接原地构造
```

Abseil Tip #112的建议: 默认使用push_back

"The less we use our power, the greater it will be." — Thomas Jefferson

核心理由:
- push_back只调用隐式构造函数 → 更安全
- emplace_back可以调用explicit构造函数 → 可能意外构造出非预期对象
- 避免"优化"降低代码安全性和清晰度，除非性能基准确认有显著收益

```cpp
// emplace_back的陷阱: 绕过explicit
std::vector<std::regex> regexes;
regexes.push_back(nullptr);         // 编译错误(good! regex(const char*)是explicit)
regexes.emplace_back(nullptr);      // 编译通过，运行时崩溃!
```

C++17新增: emplace_back返回新元素的引用(push_back返回void)，可用于链式操作。

陷阱:
- vector的push_back/emplace_back可能触发重新分配(reallocation) → 所有迭代器/引用/指针失效
- 不要无脑用emplace_back替换push_back: 安全性 > 微小性能差异

---

## 7. 智能指针

信号: 管理动态分配对象的生命周期，避免内存泄漏。

### 7.1 unique_ptr vs shared_ptr的Deleter设计差异

- unique_ptr: Deleter是模板类型参数 → 不同deleter产生不同类型
  ```cpp
  std::unique_ptr<T, MyDeleter> p1;  // 类型含deleter信息
  std::unique_ptr<T, OtherDel> p2;   // 不同类型!
  ```
- shared_ptr: Deleter不是模板参数 → 类型擦除(type erasure)
  ```cpp
  std::shared_ptr<T> p1(ptr, MyDeleter{});  // 类型相同
  std::shared_ptr<T> p2(ptr, OtherDel{});   // 类型相同!
  ```

### 7.2 void* vs char* vs uint8_t*

历史演变:
- void*出现前(1989 ANSI C之前): `char*`是通用指针，malloc返回`char*`
- void*引入后: `void*`成为通用指针，可安全赋给除函数指针外的任何指针类型
- uint8_t*: C99引入，但并非所有平台CHAR_BIT=8，所以不适合作通用指针

陷阱:
- Abseil #126: `make_unique`是新的`new` — 优先用`make_unique`而非裸`new`
  ```cpp
  auto p = std::make_unique<Widget>(arg1, arg2);  // 异常安全、简洁
  ```
- Abseil #55: unique_ptr不可拷贝，只能移动。"名字计数"规则: 变量名出现次数 = 拷贝次数(但unique_ptr阻止这一行为)
- Abseil #188: 智能指针作为函数参数要谨慎:
  - `const T&`: 只需要读值时(最常用)
  - `T*`: 可选参数(可能为null)
  - `unique_ptr<T>`: 转移所有权
  - `shared_ptr<T>`: 共享所有权
  - 不要用`const shared_ptr<T>&`传"我不需要所有权"的语义 — 用`const T&`或`T*`

---

## 8. const_cast的诡异现象

信号: 看到const_cast修改const对象的值，却发现原变量的输出没变。

机制: 常量折叠(constant folding) + 未定义行为(UB)。

```cpp
const int x = 3;
const int* y = &x;
int* z = const_cast<int*>(y);
cout << &x << ' ' << y << ' ' << z << endl;
// 三个地址相同: 0x7fffec7a5abc

*z = 4;
cout << x << ' ' << *y << ' ' << *z << endl;
// 输出: 3 4 4
// 三个指针指向同一地址，但x仍输出3!
```

原因:
1. 编译器知道x是`const int = 3`，在编译期将所有`x`的使用替换为字面量3(常量折叠)
2. 通过const_cast修改const对象是标准定义的UB
3. `*y`和`*z`在运行时读实际内存地址(已被修改为4)

cppreference明确说明:
> Modifying a const object through a non-const access path results in undefined behavior.

陷阱:
- 永远不要通过const_cast修改真正的const对象
- const_cast的合法用途: 调用只接受非const参数但不会修改的遗留API

---

## 9. 数组

信号: 传递数组给函数、理解数组退化、动态分配多维数组。

### 9.1 数组引用(不退化)

```cpp
using IntArray4 = int(&)[4];

void foo(IntArray4 array) {
    for (int i : array)          // range-based for可用(因为知道大小)
        cout << i << " ";
    cout << sizeof(array) << endl;  // 16 (= 4 * sizeof(int))，不是指针大小
    array[0] = 5;                    // 修改原数组
}
```

关键: sizeof返回整个数组大小(不退化)，可用range-based for。

### 9.2 多维数组4种堆分配方式

| 方式 | 连续性 | 复杂度 | 适用场景 |
|------|--------|--------|---------|
| 线性malloc + 规划 | 完全连续 | 中 | 需要连续内存(MPI/DMA) |
| 一维 + 下标数学 | 完全连续 | 低 | 最简单最推荐 |
| 循环malloc | 不连续(锯齿) | 中 | 每行长度不同 |
| 指针数组 | 不连续 | 低 | 简单但浪费指针空间 |

方式2(推荐):
```c
#define idx(i, j) (i * ny + j)
int* arr = malloc(nx * ny * sizeof *arr);
arr[idx(i, j)] = value;
free(arr);
```

### 9.3 2D数组传参陷阱

```c
void f(int arr[][5], int rows);    // 1. 合法: array notation
void f(int (*arr)[5], int rows);   // 2. 合法: 指向int[5]的指针(不退化)
void f(int *arr[5], int rows);     // 3. 错误! 5个int*的数组，不是二维数组
```

区分: `int (*arr)[5]` 是指向5个int的数组的指针; `int *arr[5]` 是5个int*的数组。

### 9.4 a与&a的关系

```c
int a[2] = {1, 2};
printf("a = %p\n", a);           // 00405004  (首元素地址)
printf("&a = %p\n", &a);         // 00405004  (整个数组地址)
printf("a + 1 = %p\n", a + 1);   // 00405008  (跳1个int = 4字节)
printf("&a + 1 = %p\n", &a + 1); // 0040500C  (跳整个数组 = 8字节)
```

数组名是指针常量(不可修改)，但sizeof和&运算符不触发退化。这正是C90的三个例外中的两个。

陷阱:
- 二维数组传参时最外层维度可省(编译器需知道内层维度以计算步长)
- 动态分配连续二维数组后，释放顺序是后申请的先释放
- 数组不能直接赋值(`arr1 = arr2` 非法)，但包在struct里可以

---

## 10. 不透明指针与Pimpl

信号: 需要隐藏实现细节、减少编译依赖、维护ABI稳定性。

### 10.1 C不透明指针

```c
// device.h — 只暴露前向声明
struct Device;  // incomplete type
Device* device_create(int id);
void device_destroy(Device* dev);

// device.c — 定义完整类型
struct Device { int id; int fd; void* buf; };
```

修改struct Device的内部不需要重新编译使用device.h的模块。

### 10.2 C++ Pimpl(编译防火墙)

```cpp
// PublicClass.h
class PublicClass {
public:
    PublicClass();
    PublicClass(const PublicClass&);             // 拷贝构造
    PublicClass(PublicClass&&);                  // 移动构造
    PublicClass& operator=(const PublicClass&);  // 拷贝赋值
    PublicClass& operator=(PublicClass&&);       // 移动赋值
    ~PublicClass();                              // 必须在cpp中定义
private:
    struct CheshireCat;                          // 前向声明
    std::unique_ptr<CheshireCat> d_ptr_;         // 不透明指针
};

// PublicClass.cpp
struct PublicClass::CheshireCat {
    int a; int b;
};
PublicClass::PublicClass()
    : d_ptr_(std::make_unique<CheshireCat>()) {}
PublicClass::PublicClass(const PublicClass& other)
    : d_ptr_(std::make_unique<CheshireCat>(*other.d_ptr_)) {}
PublicClass::PublicClass(PublicClass&&) = default;
PublicClass& PublicClass::operator=(const PublicClass& other) {
    *d_ptr_ = *other.d_ptr_;
    return *this;
}
PublicClass& PublicClass::operator=(PublicClass&&) = default;
PublicClass::~PublicClass() = default;
```

为什么析构函数必须在cpp中定义: unique_ptr析构时需要看到完整类型的析构函数定义。在头文件中编译器看到的是不完整类型 → 编译错误。

别名: 桥接模式、句柄类、编译器防火墙、d指针、柴郡猫。
指针 = 微笑，实现 = 身体 — 柴郡猫比喻。

### 10.3 Qt中的D-Pointer

Qt使用d-pointer的主要原因是二进制兼容性: 库的新版本可以添加私有数据成员而不破坏ABI。Qt最初是闭源的，这一设计尤其重要。

Qt的`Q_D`和`Q_Q`宏:
- `Q_D(ClassName)`: 获取d_ptr的类型安全指针
- `Q_Q(ClassName)`: 从Impl内部访问外部类

### 10.4 前向声明决策

- 指针/引用 → 前向声明足够
- 值成员/继承/sizeof → 必须#include
- 不完整类型: 只能声明指针/引用，不能实例化、访问成员、做sizeof

万恶的`struct tag`: C中用`typedef struct tag_X { ... } X;`命名结构体时，如果typedef名与tag名不同，会阻碍前向声明的可能性。正确做法:
```c
typedef struct T_Cell {   // tag名和typedef名相同
    WORD16 wCellId;
} T_Cell;
// 可以前向声明: struct T_Cell;
```

---

## 11. Abseil Tips精华(值语义与生命周期相关)

### #77 临时对象、移动与拷贝

核心: 理解临时对象何时被创建、何时被移动、何时被拷贝。现代C++编译器积极执行copy elision，很多看似需要移动的场景实际上零开销。

### #101 返回值、引用与生命周期

核心: 返回局部变量的引用是悬空引用(dangling reference)。返回值(by value)是安全的，编译器会做RVO/NRVO。

### #107 引用生命周期延长

核心: const引用可以延长临时对象的生命周期至引用的作用域结束。但这不适用于成员初始化列表中的引用成员。

### #117 Copy Elision与按值传递

核心: C++17起，prvalue的copy elision是强制的(guaranteed)。按值传递 + move到成员是一种常见且高效的模式:
```cpp
class Widget {
    std::string name_;
public:
    Widget(std::string name) : name_(std::move(name)) {}
    // 调用者传lvalue: 1次拷贝 + 1次move
    // 调用者传rvalue: 0次拷贝 + 1次move (+ copy elision)
};
```

### #176 优先返回值而非输出参数

核心: 函数应该返回值(return by value)而不是通过输出参数(指针/引用参数)返回结果。现代C++的copy elision使返回值几乎零开销。
```cpp
// BAD
void GetResult(Result* output);
// GOOD
Result GetResult();
```

### #180 避免悬空引用

核心: 引用绑定到临时对象后，如果临时对象被销毁，引用就悬空了。典型陷阱:
```cpp
const string& name = obj.GetName();  // 如果GetName()返回string(值)，临时对象活到;
                                     // 但如果是通过某个中间函数...
```

### #131 特殊成员函数与= default

核心: 用`= default`显式声明特殊成员函数(拷贝/移动构造、拷贝/移动赋值、析构)，让编译器生成默认实现。好处: 意图清晰、避免隐式删除。

Rule of Zero: 如果类不直接管理资源(用智能指针等RAII包装)，就不需要自定义任何特殊成员函数。

---

## 12. cv限定补充

### mutable

信号: 需要在const成员函数中修改某个数据成员(通常是缓存、计数器、锁)。

```cpp
class X {
    bool m_flag;
    mutable int m_accessCount;  // 允许在const函数中修改
public:
    bool GetFlag() const {
        m_accessCount++;          // 合法: mutable成员
        return m_flag;
    }
};
```

### const/constexpr

两个作用:
1. 强制思考对象的初始化和生命周期 → 影响性能
2. 向读者传达意图

如果是`static const`，编译器可以把它移到二进制文件的常量区 → 影响优化器。

### volatile

volatile禁止编译器优化掉对变量的读写。用于:
- 内存映射I/O
- 信号处理中的共享变量
- 注意: volatile不提供原子性或线程安全
