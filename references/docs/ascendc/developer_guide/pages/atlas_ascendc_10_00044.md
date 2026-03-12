# SIMD BuiltIn关键字和API-语言扩展层-编程指南-Ascend C算子开发-算子开发-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlas_ascendc_10_00044
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/opdevg/Ascendcopdevg/atlas_ascendc_10_00044.html
---

# SIMD BuiltIn关键字和API

#### 预定义宏

如其他语言一样，会提供一些内置的宏方便用户编写程序。预定义宏一节着重介绍一些用户在做异构编程时会经常用到的宏，以及宏的解释。

- __NPU_ARCH____NPU_ARCH__是Device侧AI Core代码中的预处理宏，用于标识AI处理器的架构版本。该宏由四位数字组成，其中前三位数字用于标识AI Core的IP核(Intellectual Property Core)类型，第四位数字标识该AI Core同一个IP核的配置版本。通过该宏，开发者可以针对不同AI处理器，差异化进行代码适配和优化。昇腾AI处理器型号和__NPU_ARCH__对应关系如下表所示：表1昇腾AI处理器型号和__NPU_ARCH__的对应关系昇腾AI处理器型号__NPU_ARCH__Atlas A3 训练系列产品/Atlas A3 推理系列产品2201Atlas A2 训练系列产品/Atlas A2 推理系列产品2201Atlas 200I/500 A2 推理产品3002Atlas推理系列产品2002Atlas训练系列产品1001以下为通过__NPU_ARCH__控制在不同AI处理器上算子输出值舍入模式的示例。123456789101112__aicore__staticinlinevoidCopyOut(uint64_tmulLen){#if __NPU_ARCH__ == 2002Cast(dstLocal,srcLocal,RoundMode:CAST_NONE,mulLen);// CAST_NONE表示舍入模式在转换有精度损失时使用CAST_RINT模式，在不涉及精度损失时不进行舍入#elif __NPU_ARCH__ == 2201Cast(dstLocal,srcLocal,RoundMode:CAST_RINT,mulLen);// CAST_RINT表示舍入模式为四舍六入五成双舍入#endifevent_teventVToMTE3=static_cast<event_t>(GetTPipePtr()->FetchEventID(HardEvent:V_MTE3));SetFlag<HardEvent:V_MTE3>(eventVToMTE3);WaitFlag<HardEvent:V_MTE3>(eventVToMTE3);CommonCopyOut<float>(dstLocal,mulLen);// 拷贝LocalTensor至GlobalTensor}
- ASCEND_IS_AIV、ASCEND_IS_AICASCEND_IS_AIV和ASCEND_IS_AIC是通过C++宏实现的条件判断语句，用于在__aicore__修饰的函数中实现代码的条件编译。基于分离模式（AIC核和AIV核分离）开发融合算子时，算子逻辑中同时涉及AIV核和AIC核的处理逻辑，并需要进行核间同步，此时需要通过ASCEND_IS_AIV/ ASCEND_IS_AIC进行AIV和AIC核代码的隔离。当使用高阶API Matmul时，其内部已通过REGIST_MATMUL_OBJ宏方式实现了AIV与AIC核代码的隔离，用户无需再使用该宏进行处理。以MatmulNzCustom算子为例，该算子在分离模式下需要分别在AIV核和AIC核上实现不同的逻辑。具体而言，AIV核负责将矩阵数据搬入Unified Buffer，完成数据的重排（将矩阵数据转换为NZ格式），并将其写入Global Memory。而AIC核则直接从Global Memory读取已经重排好的NZ格式数据，并执行矩阵乘法(Matmul)计算。由于AIV核和AIC核的代码逻辑不同，需要通过ASCEND_IS_AIV和ASCEND_IS_AIC宏进行代码隔离，确保在编译时分别生成适用于AIV核和AIC核的代码。示例伪码如下：12345678910111213141516171819202122232425262728293031323334template<typenameAType,typenameBType,typenameCType,typenameBiasType>__aicore__inlinevoidMatmulKernel<AType,BType,CType,BiasType>:Process(AscendC:TPipe*pipe){// 利用AIV核的Vector计算单元实现ND2NZ格式转换。如下代码中MatrixBtoNZ为将B矩阵进行ND2NZ格式转换的函数。ifASCEND_IS_AIV{pipe->InitBuffer(ubBuf,TOTAL_UB_SIZE);MatrixBtoNZ<typenameB_TYPE:T>(tempGM,bGMNZ,tiling,isTransB,ubBuf,tiling.baseK,tiling.baseN);// Vector侧实现的ND2NZ函数SyncAll();// AIC核和AIV核同步AscendC:CrossCoreSetFlag<0x2,PIPE_MTE3>(0x4);return;}ifASCEND_IS_AIC{AscendC:CrossCoreWaitFlag(0x4);// 等待AIV核完成ND2NZ格式转换}......// 设置左矩阵A、右矩阵B、Bias。matmulObj.SetTail(tailM,tailN);matmulObj.SetTensorA(aGlobal,false);matmulObj.SetTensorB(bGlobal,false);if(tiling.isBias){matmulObj.SetBias(biasGlobal);}// 完成矩阵乘操作matmulObj.IterateAll(cGlobal);// 结束矩阵乘操作matmulObj.End();}
- ASCENDC_CUBE_ONLYASCENDC_CUBE_ONLY是通过C++宏实现的条件判断语句，用于在__aicore__修饰的函数中实现代码的条件编译。基于分离模式开发非融合算子时，在只有矩阵计算的算子场景下，可以通过设置ASCENDC_CUBE_ONLY，使能纯Cube模式完成Matmul计算，减少消息通信的性能开销，提升算子性能。ASCENDC_CUBE_ONLY宏必须在#include "lib/matmul_intf.h"之前设置。以matmul_custom算子为例，高阶API Matmul默认使用MIX模式，即用户从AIV侧发起消息，通过消息通信框架中转消息后，在AIC侧执行Matmul计算。这套消息处理机制会带来额外的Scalar性能开销。相较于MIX模式，纯Cube模式可以直接跳过消息通信框架，完成Matmul计算，提升算子性能。示例伪码如下：12345678#define ASCENDC_CUBE_ONLY#include"lib/matmul_intf.h"usingA_TYPE=AscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,AType>;usingB_TYPE=AscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,BType>;usingC_TYPE=AscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,CType>;usingBIAS_TYPE=AscendC:MatmulType<AscendC:TPosition:GM,CubeFormat:ND,BiasType>;AscendC:Matmul<A_TYPE,B_TYPE,C_TYPE,BIAS_TYPE,CFG_NORM>matmulObj;

#### 函数执行空间限定符

函数执行空间限定符(Function Execution Space Qualifier)指示函数是在Host侧执行还是在Device侧执行，以及它是否可从Host侧或Device侧调用。

- __global____global__执行空间限定符声明一个Kernel函数。Kernel函数有如下性质：在Device上执行；只能被Host侧函数调用；__global__只是表示这是Device侧函数的入口，并不表示具体的设备类型，具体的设备类型由__aicore__标记。具有如下使用约束：一个__global__函数必须返回void类型，并且不能是class的成员函数。主机侧调用__global__函数必须使用<<<>>>异构调用语法。__global__的调用是异步的，意味着函数返回，并不表示kernel函数在device侧已经执行完成，如果需要同步，需要使用Runtime同步接口显式同步，如aclrtSynchronizeStream接口。
- __aicore____aicore__执行空间限定符声明一个函数，它具有如下属性：在Device侧执行只能被__global__函数，或者其他__aicore__函数调用// Only callable from device functions with same kind
// of execution space
__aicore__ void bar() {}

// Define a kernel function execute on AI Core device
__global__ __aicore__ void foo() {
  bar(); // OK.
}

- __host____host__执行空间限定符声明一个函数，它具有如下属性：只能在Host侧执行只能被Host侧函数调用__global__和__host__不能一起使用__host__限定符是可选项，无函数执行空间限定符定义的函数，默认是host函数。__aicore__ int f() {}

// defines a host side function
int foo() {}

// defines a host side function
__host__ int bar() {
  f();     // Error.
  foo();   // OK.
}

// Error.
__global__ __host__ void kfunc() {}

- __inline____inline__限定符声明一个函数，它具有如下属性：标识Device侧函数强制内联，可以减少函数频繁调用产生的指令压栈、出栈的开销，但可能会导致算子二进制增加。和C++函数修饰符inline的主要区别是Device侧__inline__是强制内联，C++的inline则是根据编译器优化选择性内联。AI Core对函数嵌套深度有限制，一般推荐嵌套深度不超过4层。使用强制内联可以减少调用层次。

- __cube__标识该核函数仅在Cube核执行。针对耦合模式的硬件架构，该修饰符不生效。__vector__ __global__ __aicore__ void add_custom(){}

- __vector__标识该核函数仅在Vector核执行。针对耦合模式的硬件架构，该修饰符不生效。

- __mix__(cube, vec)标识该核函数同时在Cube核和Vector核上执行。(cube, vec)分别表示核函数启动的Cube核和Vector核的配比，支持的配比为(1, 0)，(0, 1)，(1, 1)， (1, 2)。针对耦合模式的硬件架构，该修饰符不生效。

#### 地址空间限定符

AI Core具备多级独立片上存储，各个地址空间独立编址，具备各自的访存指令，根据架构差异，有些存储空间具备统一地址空间(Generic Address Space)，有些则没有。设备侧编程基于语法扩展允许地址空间作为合法的类型限定符，以提供针对不同地址空间的访问能力和地址空间合法性检查。

| 地址空间限定符 | AI Core物理存储空间   |
| -------------- | --------------------- |
| __gm__         | 设备侧内存GM          |
| __ubuf__       | Vector Unified Buffer |
| __ca__         | Cube L0A Buffer       |
| __cb__         | Cube L0B Buffer       |
| __cc__         | Cube L0C Buffer       |
| __cbuf__       | Cube L1 Buffer        |
| __fbuf__       | Fixpipe Buffer        |

地址空间限定符可以在变量声明中使用，用于指定对象分配的区域。如果对象的类型被地址空间名称限定，那么该对象将被分配在指定的地址空间中。同样地，对于指针，指向的类型可以通过地址空间进行限定，以指示所指向的对象所在的地址空间。

地址空间限定符不能用于非指针返回类型，非指针函数参数，函数类型，同一个类型上不允许使用多个地址空间限定符。

![](../images/atlas_ascendc_10_00043_img_002.png)

重要：不同地址空间指针的大小可能不同。例如，不能认为sizeof(__gm__ int *)总是等于sizeof(__ubuf__ int *)，譬如编译器或许可能在某些系统上以32bit存储__ubuf__指针。

- private地址空间private地址空间是大多数变量的默认地址空间，特别是局部变量。// m is in a specific kernel parameter address space, 
// it's physical location is implementation determined.
__global__ __aicore__ void foo(int m) {
  // OK. i is an int variable allocated in private address space
  int i;
}

__aicore__ void bar(int k) { //OK. k is in private address space
  // OK. i is an int variable allocated in private address space
  int i; 
}

- __gm__地址空间__gm__地址空间限定符用来表示分配于设备侧全局内存的对象，全局内存对象可以声明为标量、用户自定义结构体的指针。__gm__ int *var; // var point to an array of int elements

typedef struct {
    float a[3];
    int b[2];
} foo_t;

__gm__ foo_t *info; // info point to an array of foo_t elements

- __ubuf__地址空间__ubuf__地址空间用来描述存储于AI Core核内UB存储空间的变量。__global__ __aicore__ void foo() {
  // ptr is in private address space, point to __ubuf__
  __ubuf__ int *ptr;
}

- __ca__, __cb__, __cc__, __cbuf__地址空间上述几个地址空间主要用于特定的DMA指令访问，不具备标量直接访问能力。class ObjTy{
  ObjTy(){...}
  void print(){...}

private:
  int a;
  int b;
};

__global__ __aicore__ 
void foo(__ca__ int * ptr) { // Error. Cannot have __ca__
                             // qualifier in kernel arguments
  // OK
  __ca__ int *ptr; 
}

#### 内置常量

| 常量名                       | 取值                   | 功能                                                                                                                                                                                         |
| ---------------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| constexpr int32_t g_coreType | AscendC:AICAscendC:AIV | 常量值由框架自动设置，AIC核下，配置为AscendC:AIC，AIV核下，配置为AscendC:AIV。可以通过对该常量值的判断，来实现了AIV与AIC核代码的区分和隔离。功能等同于直接使用ASCEND_IS_AIV、ASCEND_IS_AIC。 |

#### 内置变量

| 变量名    | 对应API     | 功能                                                         |
| --------- | ----------- | ------------------------------------------------------------ |
| block_num | GetBlockNum | 当前任务配置的核数，用于代码内部的多核逻辑控制等。           |
| block_idx | GetBlockIdx | 当前核的索引，用于代码内部的多核逻辑控制及多核偏移量计算等。 |

通常，建议用户使用内置变量对应的API获取所需值，不建议用户直接使用内置变量。因为内置变量反映的是单个硬件资源的配置信息，对于软件栈整合硬件资源、扩展硬件的功能，内置变量的值与实际语义可能不符。

例如，在Atlas推理系列产品中，当启用KERNEL_TYPE_MIX_VECTOR_CORE时，算子会同时运行在AI Core和Vector Core上。此时，block_idx在这两种核心上都是从0开始计数，用户无法直接通过block_idx来切分数据和控制多核逻辑。而GetBlockIdx在Vector Core上对block_idx增加偏移量（AI Core的block_num），从而保证返回的值能够正确反映多核环境下的实际逻辑。

#### BuiltIn API

具体API列表请参见《CCE Intrinsic开发接口》。
