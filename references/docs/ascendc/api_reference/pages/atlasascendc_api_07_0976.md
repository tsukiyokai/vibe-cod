# OpAttrDef-OpAttrDef-原型注册与管理-Utils API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0976
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0976.html
---

# OpAttrDef

#### 功能说明

定义算子属性。

#### 函数原型

| 1234567891011121314151617181920212223242526272829 | classOpAttrDef{public:explicitOpAttrDef(constchar*name);OpAttrDef(constOpAttrDef&attr_def);~OpAttrDef();OpAttrDef&operator=(constOpAttrDef&attr_def);OpAttrDef&AttrType(Optionattr_type);OpAttrDef&Bool(void);OpAttrDef&Bool(boolvalue);OpAttrDef&Float(void);OpAttrDef&Float(floatvalue);OpAttrDef&Int(void);OpAttrDef&Int(int64_tvalue);OpAttrDef&String(void);OpAttrDef&String(constchar*value);OpAttrDef&ListBool(void);OpAttrDef&ListBool(std:vector<bool>value);OpAttrDef&ListFloat(void);OpAttrDef&ListFloat(std:vector<float>value);OpAttrDef&ListInt(void);OpAttrDef&ListInt(std:vector<int64_t>value);OpAttrDef&ListListInt(void);OpAttrDef&ListListInt(std:vector<std:vector<int64_t>>value);OpAttrDef&Version(uint32_tversion);ge:AscendString&GetName(void)const;boolIsRequired(void);private:...}; |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 函数说明

| 函数名称    | 入参说明              | 功能说明                                                                                                                                                                                                                                                |
| ----------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AttrType    | attr_type: 属性类型   | 设置算子属性类型，取值为：OPTIONAL（可选）、REQUIRED（必选）。                                                                                                                                                                                          |
| Bool        | 无                    | 设置算子属性数据类型为Bool                                                                                                                                                                                                                              |
| Bool        | value                 | 设置算子属性数据类型为Bool，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                               |
| Float       | 无                    | 设置算子属性数据类型为Float                                                                                                                                                                                                                             |
| Float       | value                 | 设置算子属性数据类型为Float，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                              |
| Int         | 无                    | 设置算子属性数据类型为Int                                                                                                                                                                                                                               |
| Int         | value                 | 设置算子属性数据类型为Int，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                                |
| String      | 无                    | 设置算子属性数据类型为String                                                                                                                                                                                                                            |
| String      | value                 | 设置算子属性数据类型为String，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                             |
| ListBool    | 无                    | 设置算子属性数据类型为ListBool                                                                                                                                                                                                                          |
| ListBool    | value                 | 设置算子属性数据类型为ListBool，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                           |
| ListFloat   | 无                    | 设置算子属性数据类型为ListFloat                                                                                                                                                                                                                         |
| ListFloat   | value                 | 设置算子属性数据类型为ListFloat，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                          |
| ListInt     | 无                    | 设置算子属性数据类型为ListInt                                                                                                                                                                                                                           |
| ListInt     | value                 | 设置算子属性数据类型为ListInt，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                            |
| ListListInt | 无                    | 设置算子属性数据类型为ListListInt                                                                                                                                                                                                                       |
| ListListInt | value                 | 设置算子属性数据类型为ListListInt，并设置属性默认值为value。属性类型设置为OPTIONAL时必须调用该类接口设置默认值。                                                                                                                                        |
| Version     | version：配置的版本号 | 新增可选属性时，为了保持原有单算子API(aclnnxxx)接口的兼容性，可以通过Version接口配置aclnn接口的版本号，版本号需要从1开始配，且应该连续配置（和可选输入统一编号）。配置后，自动生成的aclnn接口会携带版本号。高版本号的接口会包含低版本号接口的所有参数。 |
| GetName     | 无                    | 获取属性名称。                                                                                                                                                                                                                                          |
| IsRequired  | 无                    | 判断算子属性是否为必选，必选返回true，可选返回false。                                                                                                                                                                                                   |
