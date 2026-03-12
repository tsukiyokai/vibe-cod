# is_base_of-类型特性-C++标准库-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10115
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10115.html
---

# is_base_of

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

is_base_of是定义于<type_traits>头文件的一个类型特征工具，它能够在程序编译时检查一个类型是否为另一个类型的基类。本接口可应用在模板元编程、类型检查和条件编译等场景，用于在编译阶段捕获潜在的类型错误，提高代码的鲁棒性。

#### 函数原型

| 12  | template<typenameBase,typenameDerived>structis_base_of; |
| --- | ------------------------------------------------------- |

#### 参数说明

| 参数名  | 含义                                                    |
| ------- | ------------------------------------------------------- |
| Base    | 待检查的基类类型，即Base类型是否为Derived类型的基类。   |
| Derived | 待检查的派生类类型，即Base类型是否为Derived类型的基类。 |

#### 约束说明

无

#### 返回值说明

is_base_of的静态常量成员value用于获取返回的布尔值，is_base_of<Base, Derived>:value取值如下：

- true：Base类型是Derived类型的基类（包括Base类型和Derived类型为同一类型的情况）。
- false：Base类型不是Derived类型的基类。

#### 调用示例

| 1234567891011121314151617181920212223242526272829303132333435363738394041 | classBase{};classDerived:publicBase{};classUnrelated{};// 虚继承的派生类classDerived2:virtualpublicBase{};// 定义虚继承的派生类classVirtualDerived:virtualpublicBase{};// 定义多重继承的派生类classMultiDerived:publicBase,publicVirtualDerived{};// 模板基类template<typenameT>classBaseTemplate{public:Tvalue;};// 模板派生类template<typenameT>classDerivedTemplate:publicBaseTemplate<T>{};// 检查Base是否是Derived的基类AscendC:PRINTF("Is Base a base of Derived? %d\n",AscendC:Std:is_base_of<Base,Derived>:value);// 检查Derived是否是Base的基类（应该为false）AscendC:PRINTF("Is Derived a base of Base? %d\n",AscendC:Std:is_base_of<Derived,Base>:value);// 检查Base是否是Unrelated的基类（应该为false）AscendC:PRINTF("Is Base a base of Unrelated? %d\n",AscendC:Std:is_base_of<Base,Unrelated>:value);AscendC:PRINTF("Is Base a base of Derived (virtual inheritance)? %d\n",AscendC:Std:is_base_of<Base,Derived2>:value);AscendC:PRINTF("Is BaseTemplate<int> a base of DerivedTemplate<int>? %d\n",AscendC:Std:is_base_of<BaseTemplate<int>,DerivedTemplate<int>>:value);// 测试Base是否为VirtualDerived的基类（虚继承情况）AscendC:PRINTF("Is Base a base of VirtualDerived? %d\n",AscendC:Std:is_base_of<Base,VirtualDerived>:value);// 测试Base是否为MultiDerived的基类（多重继承情况）AscendC:PRINTF("Is Base a base of MultiDerived? %d\n",AscendC:Std:is_base_of<Base,MultiDerived>:value); |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 12345678 | // 执行结果：IsBaseabaseofDerived?1IsDerivedabaseofBase?0IsBaseabaseofUnrelated?0IsBaseabaseofDerived(virtualinheritance)?1IsBaseTemplate<int>abaseofDerivedTemplate<int>?1IsBaseabaseofVirtualDerived?1IsBaseabaseofMultiDerived?1 |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
