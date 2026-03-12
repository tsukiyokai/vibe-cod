# conditional-类型特性-C++标准库-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10118
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10118.html
---

# conditional

#### 产品支持情况

| 产品                                        | 是否支持 |
| ------------------------------------------- | -------- |
| Atlas A3 训练系列产品/Atlas A3 推理系列产品 | √        |
| Atlas A2 训练系列产品/Atlas A2 推理系列产品 | √        |
| Atlas 200I/500 A2 推理产品                  | x        |
| Atlas推理系列产品AI Core                    | x        |
| Atlas推理系列产品Vector Core                | x        |
| Atlas训练系列产品                           | x        |

#### 功能说明

conditional是定义在<type_traits>头文件里的一个类型特征工具，它在程序编译时根据一个布尔条件从两个类型中选择一个类型。本接口可应用在模板元编程中，用于根据不同的条件来灵活选择合适的类型，增强代码的通用性和灵活性。

conditional有一个嵌套的type成员，它的值取决于Bp的值：如果Bp为true，则conditional<Bp, If, Then>:type为If。如果Bp为false，则conditional<Bp, If, Then>:type为Then。

#### 函数原型

| 12  | template<boolBp,typenameIf,typenameThen>structconditional; |
| --- | ---------------------------------------------------------- |

#### 参数说明

| 参数名 | 含义                                     |
| ------ | ---------------------------------------- |
| Bp     | 一个布尔常量表达式，作为选择类型的条件。 |
| If     | 当Bp为true时选择的类型。                 |
| Then   | 当Bp为false时选择的类型。                |

#### 约束说明

无

#### 返回值说明

conditional的静态常量成员type用于获取返回值，conditional<Bp, If, Then>:type取值如下：

- If：Bp为true。
- Then：Bp为false。

#### 调用示例

| 12345678910111213141516171819202122232425262728293031323334353637383940 | // 定义两个不同的类型structTypeA{__aicore__inlinestaticvoidprint(){AscendC:PRINTF("This is TypeA..\n");}};structTypeB{__aicore__inlinestaticvoidprint(){AscendC:PRINTF("This is TypeB..\n");}};// 根据条件选择类型template<boolCondition>__aicore__inlinevoidselectType(){usingSelectedType=typenameAscendC:Std:conditional<Condition,TypeA,TypeB>:type;SelectedType:print();}// 定义一个模板函数，根据条件选择不同的类型template<boolCondition>__aicore__inlinevoidselectOtherType(){usingSelectedType=typenamestd:conditional<Condition,int,float>:type;ifconstexpr(std:is_same_v<SelectedType,int>){AscendC:PRINTF("Selected type is int.\n");}else{AscendC:PRINTF("Selected type is float.\n");}}// 条件为true，选择TypeAselectType<true>();// 条件为false，选择TypeBselectType<false>();// 测试条件为true的情况selectOtherType<true>();// 测试条件为false的情况selectOtherType<false>(); |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12345 | // 执行结果：ThisisTypeA..ThisisTypeB..Selectedtypeisint.Selectedtypeisfloat. |
| ----- | ----------------------------------------------------------------------------- |
