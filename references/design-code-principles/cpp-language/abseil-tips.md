# Abseil C++ Tips of the Week

https://abseil.io/tips/

## 字符串

| # | 标题 | 链接 |
|---|------|------|
| 1 | `string_view` | https://abseil.io/tips/1 |
| 3 | String Concatenation and `operator+` vs. `StrCat()` | https://abseil.io/tips/3 |
| 10 | Splitting Strings, not Hairs | https://abseil.io/tips/10 |
| 18 | String Formatting with Substitute | https://abseil.io/tips/18 |
| 36 | New Join API | https://abseil.io/tips/36 |
| 59 | Joining Tuples | https://abseil.io/tips/59 |
| 64 | Raw String Literals | https://abseil.io/tips/64 |
| 124 | `absl::StrFormat()` | https://abseil.io/tips/124 |
| 215 | Stringifying Custom Types with `AbslStringify()` | https://abseil.io/tips/215 |

## 对象生命周期与移动语义

| # | 标题 | 链接 |
|---|------|------|
| 5 | Disappearing Act | https://abseil.io/tips/5 |
| 11 | Return Policy | https://abseil.io/tips/11 |
| 24 | Copies, Abbrv. | https://abseil.io/tips/24 |
| 55 | Name Counting and `unique_ptr` | https://abseil.io/tips/55 |
| 65 | Putting Things in their Place | https://abseil.io/tips/65 |
| 77 | Temporaries, Moves, and Copies | https://abseil.io/tips/77 |
| 101 | Return Values, References, and Lifetimes | https://abseil.io/tips/101 |
| 107 | Reference Lifetime Extension | https://abseil.io/tips/107 |
| 116 | Keeping References on Arguments | https://abseil.io/tips/116 |
| 117 | Copy Elision and Pass-by-value | https://abseil.io/tips/117 |
| 120 | Return Values are Untouchable | https://abseil.io/tips/120 |
| 126 | `make_unique` is the new `new` | https://abseil.io/tips/126 |
| 134 | `make_unique` and `private` Constructors | https://abseil.io/tips/134 |
| 149 | Object Lifetimes vs. `= delete` | https://abseil.io/tips/149 |
| 166 | When a Copy is not a Copy | https://abseil.io/tips/166 |
| 176 | Prefer Return Values to Output Parameters | https://abseil.io/tips/176 |
| 180 | Avoiding Dangling References | https://abseil.io/tips/180 |
| 187 | `std::unique_ptr` Must Be Moved | https://abseil.io/tips/187 |
| 188 | Be Careful With Smart-Pointer Function Parameters | https://abseil.io/tips/188 |

## 初始化与构造

| # | 标题 | 链接 |
|---|------|------|
| 42 | Prefer Factory Functions to Initializer Methods | https://abseil.io/tips/42 |
| 61 | Default Member Initializers | https://abseil.io/tips/61 |
| 74 | Delegating and Inheriting Constructors | https://abseil.io/tips/74 |
| 88 | Initialization: `=`, `()`, and `{}` | https://abseil.io/tips/88 |
| 112 | `emplace` vs. `push_back` | https://abseil.io/tips/112 |
| 131 | Special Member Functions and `= default` | https://abseil.io/tips/131 |
| 142 | Multi-parameter Constructors and `explicit` | https://abseil.io/tips/142 |
| 143 | C++11 Deleted Functions (`= delete`) | https://abseil.io/tips/143 |
| 146 | Default vs Value Initialization | https://abseil.io/tips/146 |
| 172 | Designated Initializers | https://abseil.io/tips/172 |
| 182 | Initialize Your Ints! | https://abseil.io/tips/182 |

## 接口设计与API风格

| # | 标题 | 链接 |
|---|------|------|
| 76 | Use `absl::Status` | https://abseil.io/tips/76 |
| 86 | Enumerating with Class | https://abseil.io/tips/86 |
| 94 | Callsite Readability and bool Parameters | https://abseil.io/tips/94 |
| 99 | Nonmember Interface Etiquette | https://abseil.io/tips/99 |
| 109 | Meaningful `const` in Function Declarations | https://abseil.io/tips/109 |
| 123 | `absl::optional` and `std::unique_ptr` | https://abseil.io/tips/123 |
| 141 | Beware Implicit Conversions to `bool` | https://abseil.io/tips/141 |
| 163 | Passing `std::optional` parameters | https://abseil.io/tips/163 |
| 171 | Avoid Sentinel Values | https://abseil.io/tips/171 |
| 173 | Wrapping Arguments in Option Structs | https://abseil.io/tips/173 |
| 177 | Assignability vs. Data Member Types | https://abseil.io/tips/177 |
| 181 | Accessing the value of a `StatusOr<T>` | https://abseil.io/tips/181 |
| 198 | Tag Types | https://abseil.io/tips/198 |
| 218 | Designing Extension Points With FTADLE | https://abseil.io/tips/218 |
| 234 | Pass by Value, by Pointer, or by Reference? | https://abseil.io/tips/234 |

## 容器与算法

| # | 标题 | 链接 |
|---|------|------|
| 93 | using `absl::Span` | https://abseil.io/tips/93 |
| 136 | Unordered Containers | https://abseil.io/tips/136 |
| 144 | Heterogeneous Lookup in Associative Containers | https://abseil.io/tips/144 |
| 152 | `AbslHashValue` and You | https://abseil.io/tips/152 |
| 158 | Abseil Associative containers and `contains()` | https://abseil.io/tips/158 |
| 224 | Avoid `vector.at()` | https://abseil.io/tips/224 |
| 227 | Be Careful with Empty Containers and Unsigned Arithmetic | https://abseil.io/tips/227 |
| 231 | Between Here and There: Some Minor Overlooked Algorithms | https://abseil.io/tips/231 |

## 命名空间与作用域

| # | 标题 | 链接 |
|---|------|------|
| 119 | Using-declarations and namespace aliases | https://abseil.io/tips/119 |
| 130 | Namespace Naming | https://abseil.io/tips/130 |
| 153 | Don't Use using-directives | https://abseil.io/tips/153 |
| 186 | Prefer to Put Functions in the Unnamed Namespace | https://abseil.io/tips/186 |

## 语言特性与惯用法

| # | 标题 | 链接 |
|---|------|------|
| 49 | Argument-Dependent Lookup | https://abseil.io/tips/49 |
| 108 | Avoid `std::bind` | https://abseil.io/tips/108 |
| 140 | Constants: Safe Idioms | https://abseil.io/tips/140 |
| 147 | Use Exhaustive `switch` Statements Responsibly | https://abseil.io/tips/147 |
| 148 | Overload Sets | https://abseil.io/tips/148 |
| 161 | Good Locals and Bad Locals | https://abseil.io/tips/161 |
| 165 | `if` and `switch` statements with initializers | https://abseil.io/tips/165 |
| 168 | `inline` Variables | https://abseil.io/tips/168 |
| 175 | Changes to Literal Constants in C++14 and C++17 | https://abseil.io/tips/175 |
| 229 | Ranked Overloads for Template Metaprogramming | https://abseil.io/tips/229 |
| 232 | When to Use `auto` for Variable Declarations | https://abseil.io/tips/232 |

## Flags

| # | 标题 | 链接 |
|---|------|------|
| 45 | Avoid Flags, Especially in Library Code | https://abseil.io/tips/45 |
| 90 | Retired Flags | https://abseil.io/tips/90 |
| 103 | Flags Are Globals | https://abseil.io/tips/103 |

## 测试

| # | 标题 | 链接 |
|---|------|------|
| 122 | Test Fixtures, Clarity, and Dataflow | https://abseil.io/tips/122 |
| 135 | Test the Contract, not the Implementation | https://abseil.io/tips/135 |

## 并发

| # | 标题 | 链接 |
|---|------|------|
| 197 | Reader Locks Should Be Rare | https://abseil.io/tips/197 |
