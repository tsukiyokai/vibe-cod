# 图编译和图执行-算子入图（GE图）开发-附录-编程指南-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_10_0079
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_10_0079.html
---

# 图编译和图执行

本节通过单算子模型执行的样例来介绍图模式下图编译和图执行流程。单算子模型执行是指基于图IR执行算子，先编译算子（例如，使用ATC工具将Ascend IR定义的单算子描述文件编译成算子om模型文件），再调用acl接口加载算子模型，最后调用acl接口执行算子。下文仅提供单算子模型执行的样例和基础内容讲解，详细内容请参考单算子模型执行。

#### 环境要求

- 已参考环境准备，完成CANN驱动和软件的安装，配置CANN软件所需基本环境变量。安装CANN软件后，使用CANN运行用户进行编译、运行时，需要以CANN运行用户登录环境，执行source ${install_path}/set_env.sh命令设置环境变量。其中${install_path}为CANN软件的安装目录，例如：/usr/local/Ascend/cann。

- 已参考工程化算子开发完成算子的开发和部署。

#### 准备验证代码工程

#### 生成单算子离线模型文件

1. 构造静态shape单算子描述文件add_custom_static_shape.json，描述算子的输入、输出及属性等信息。AddCustom静态shape算子的描述文件示例如下：[
    {
        "op": "AddCustom",
        "input_desc": [
            {
                "name": "x",
                "param_type": "required",
                "format": "ND",
                "shape": [8, 2048],
                "type": "float16"
            },
            {
                "name": "y",
                "param_type": "required",
                "format":"ND",
                "shape": [8, 2048],
                "type": "float16"
            }
        ],
        "output_desc": [
            {
                "name": "z",
                "param_type": "required",
                "format":  "ND",
                "shape": [8, 2048],
                "type": "float16"
            }
        ]
    }
]AddCustom动态shape算子的描述文件示例如下：[
    {
        "op": "AddCustom",
        "input_desc": [
            {
                "name": "x",
                "param_type": "required",
                "format": "ND",
                "shape": [-1, -1],
                "shape_range": [[1,-1],[1,-1]],
                "type": "float16"
            },
            {
                "name": "y",
                "param_type": "required",
                "format":"ND",
                "shape": [-1, -1],
                "shape_range": [[1,-1],[1,-1]],
                "type": "float16"
            }
        ],
        "output_desc": [
            {
                "name": "z",
                "param_type": "required",
                "format":  "ND",
                "shape": [-1, -1],
                "shape_range": [[1,-1],[1,-1]],
                "type": "float16"
            }
        ]
    }
]
1. 使用ATC工具，将该算子描述文件编译成单算子模型文件（*.om文件）ATC工具的命令示例如下：atc --singleop=$HOME/op_verify/run/out/test_data/config/add_custom_static_shape.json--output=op_models/ --soc_version=<soc_version>关键参数解释如下（详细参数说明，请参见《ATC离线模型编译工具用户指南》。）：--singleop：单算子描述文件（json格式）的路径。--output：存放om模型文件的目录。--soc_version：配置为AI处理器的型号，请根据实际环境进行替换。AI处理器的型号请通过如下方式获取：针对如下产品：在安装昇腾AI处理器的服务器执行npu-smi info命令进行查询，获取Name信息。实际配置值为AscendName，例如Name取值为xxxyy，实际配置值为Ascendxxxyy。Atlas A2 训练系列产品/Atlas A2 推理系列产品Atlas 200I/500 A2 推理产品Atlas推理系列产品Atlas训练系列产品针对如下产品，在安装昇腾AI处理器的服务器执行npu-smi info -t board -iid-cchip_id命令进行查询，获取Chip Name和NPU Name信息，实际配置值为Chip Name_NPU Name。例如Chip Name取值为Ascendxxx，NPU Name取值为1234，实际配置值为Ascendxxx_1234。其中：id：设备id，通过npu-smi info -l命令查出的NPU ID即为设备id。chip_id：芯片id，通过npu-smi info -m命令查出的Chip ID即为芯片id。Atlas A3 训练系列产品/Atlas A3 推理系列产品以上命令执行后，会在output参数指定路径下生成*.om后缀的离线模型文件。

#### 生成测试数据

在样例工程目录下，执行如下命令：

会在input目录下生成两个shape为(8,2048)，数据类型为float16的数据文件input_0.bin与input_1.bin，用于进行AddCustom算子的验证。

代码样例如下：

#### 编写验证代码

您可以参考如下样例编写单算子加载、执行的代码逻辑。

以下是关键步骤的代码示例，不可以直接拷贝编译运行，仅供参考，调用接口后，需增加异常处理的分支，并记录报错日志、提示日志，此处不一一列举。

| 12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152 | // 1.初始化aclRet=aclInit("../scripts/acl.json");// 2.运行管理资源申请intdeviceId=0;aclRet=aclrtSetDevice(deviceId);// 获取软件栈的运行模式，不同运行模式影响后续的接口调用流程（例如是否进行数据传输等）aclrtRunModerunMode;boolg_isDevice=false;aclErroraclRet=aclrtGetRunMode(&runMode);g_isDevice=(runMode==ACL_DEVICE);// 3.加载单算子模型文件（*.om文件）// 该目录相对可执行文件所在的目录，例如，编译出来的可执行文件存放在output目录下，此处就表示工程目录下的op_models目录aclRet=aclopSetModelDir("../op_models");// 4.设置算子的输入，申请内存，然后读取输入数据input_0.bin与input_1.bin并保存至申请的内存中// ......// 5.创建Stream流aclrtStreamstream=nullptr;aclrtCreateStream(&stream)// 6.执行算子// opType表示算子类型名称，例如AddCustom// numInputs表示算子输入个数，例如AddCustom算子是2个输入// inputDesc表示算子输入tensor描述的数组，描述每个输入的format、shape、数据类型// inputs表示算子输入tensor数据// numOutputs表示算子输出个数，例如AddCustom算子是1个输出// outputDesc表示算子输出tensor描述的数组，描述每个输出的format、shape、数据类型// outputs表示算子输出tensor数据// attr表示算子属性，如果算子没有属性，也需要调用aclopCreateAttr接口创建aclopAttr类型的数据// stream用于维护一些异步操作的执行顺序aclopExecuteV2(opType,numInputs,inputDesc,inputs,numOutputs,outputDesc,outputs,attr,nullptr);// 7.阻塞应用运行，直到指定Stream中的所有任务都完成aclrtSynchronizeStream(stream);// 8.处理执行算子后的输出数据，例如在屏幕上显示、写入文件等，由用户根据实际情况自行实现，本示例会将结果写入output_z.bin文件中// ......// 9.释放stream流aclrtDestroyStream(stream);// 10.释放运行管理资源aclRet=aclrtResetDevice(deviceId);aclRet=aclFinalize();// .... |
| ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 运行和验证

1. 开发环境上，设置环境变量，配置单算子验证程序编译依赖的头文件与库文件路径，如下为设置环境变量的示例。${INSTALL_DIR}请替换为CANN软件安装后文件存储路径。以root用户安装为例，则安装后文件存储路径为：/usr/local/Ascend/cann。{arch-os}为运行环境的架构和操作系统，arch表示操作系统架构，os表示操作系统，例如x86_64-linux。export DDK_PATH=${INSTALL_DIR}export NPU_HOST_LIB=${INSTALL_DIR}/{arch-os}/devlib
1. 编译样例工程，生成单算子验证可执行文件。切换到样例工程根目录，然后在样例工程根目录下执行如下命令创建目录用于存放编译文件，例如，创建的目录为“build”。mkdir -p build进入build目录，执行cmake编译命令，生成编译文件命令示例如下所示：cd buildcmake ../src -DCMAKE_SKIP_RPATH=TRUE执行如下命令，生成可执行文件。make会在工程目录的output目录下生成可执行文件execute_add_op。
1. 执行单算子以运行用户（例如HwHiAiUser）拷贝开发环境中样例工程output下的execute_add_op到运行环境任一目录。说明：若您的开发环境即为运行环境，此拷贝操作可跳过。在运行环境中，执行execute_add_op文件，验证单算子模型文件。chmod +x execute_add_op./execute_add_op会有如下屏显信息：12345678910111213141516[INFO]staticopwillbecalled[INFO]Setdevice[0]success[INFO]GetRunMode[1]success[INFO]aclopSetModelDiropmodelsuccess[INFO]Initresourcesuccess[INFO]Setinputsuccess[INFO]Copyinput[0]success[INFO]Copyinput[1]success[INFO]Createstreamsuccess[INFO]ExecuteAddCustomsuccess[INFO]Synchronizestreamsuccess[INFO]Copyoutput[0]success[INFO]Writeoutputsuccess[INFO]Runopsuccess[INFO]ResetDevicesuccess[INFO]Destroyresourcesuccess如果有Run op success，表明执行成功，会在output目录下生成输出文件output_z.bin。
1. 比较真值文件切换到样例工程根目录，然后执行如下命令：python3 scripts/verify_result.py output/output_z.bin output/golden.bin会有如下屏显信息，可见AddCustom算子验证结果正确。1testpass
