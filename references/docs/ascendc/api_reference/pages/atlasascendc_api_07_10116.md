# is_same-类型特性-C++标准库-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_10116
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_10116.html
---

# is_same

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

is_same是定义在<type_traits>头文件里的一个类型特征工具，它能够在程序编译时判断两个类型是否完全相同。本接口可应用在模板元编程、类型检查、条件编译等场景，用于在编译阶段确定类型信息，避免运行时可能出现的类型不匹配问题。

#### 函数原型

| 12  | template<typenameTp,typenameUp>structis_same; |
| --- | --------------------------------------------- |

#### 参数说明

| 参数名 | 含义                                       |
| ------ | ------------------------------------------ |
| Tp     | 需要比较两个类型是否完全相同的第一个类型。 |
| Up     | 需要比较两个类型是否完全相同的第二个类型。 |

#### 约束说明

无

#### 返回值说明

is_same的静态常量成员value用于获取返回的布尔值，is_same<Tp, Up>:value取值如下：

- true：Tp和Up是完全相同的类型。
- false：Tp和Up不是相同的类型。

#### 调用示例

| 1234567891011121314151617181920212223242526272829 | // 定义两个不同的类classClassA{};classClassB{};// 定义相同的类两次classClassC{};usingClassC_alias=ClassC;// 定义一个简单的模板类template<typenameT>classTemplateClass{};// 比较相同的基本类型AscendC:PRINTF("Is int the same as int? %d\n",AscendC:Std:is_same<int,int>:value);// 比较不同的基本类型AscendC:PRINTF("Is int the same as double? %d\n",AscendC:Std:is_same<int,double>:value);// 比较不同的类类型AscendC:PRINTF("Is ClassA the same as ClassB? %d\n",AscendC:Std:is_same<ClassA,ClassB>:value);// 比较相同的类类型AscendC:PRINTF("Is ClassC the same as ClassC_alias? %d\n",AscendC:Std:is_same<ClassC,ClassC_alias>:value);// 比较相同模板实例化类型AscendC:PRINTF("Is TemplateClass<int> the same as TemplateClass<int>? %d\n",AscendC:Std:is_same<TemplateClass<int>,TemplateClass<int>>:value);// 比较不同模板实例化类型AscendC:PRINTF("Is TemplateClass<int> the same as TemplateClass<double>? %d\n",AscendC:Std:is_same<TemplateClass<int>,TemplateClass<double>>:value); |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

| 1234567 | // 执行结果：Isintthesameasint?1Isintthesameasdouble?0IsClassAthesameasClassB?0IsClassCthesameasClassC_alias?1IsTemplateClass<int>thesameasTemplateClass<int>?1IsTemplateClass<int>thesameasTemplateClass<double>?0 |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
