# HCCL Tiling使用说明-HCCL Tiling侧接口-HCCL通信类-高阶API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0886
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0886.html
---

# HCCL Tiling使用说明

![](../images/atlasascendc_api_07_10005_img_003.png)

根据使用标准C++语法定义Tiling结构体的方式，Ascend C提供一组HCCL Tiling API，方便用户获取HCCL Kernel计算时所需的Tiling参数。您只需要传入通信的相关信息，调用API接口，即可获取通信相关的Tiling参数。

HCCL Tiling API获取Tiling参数的流程如下：

1. 创建一个Mc2CcTilingConfig类对象。12345constchar*groupName="testGroup";uint32_topType=HCCL_CMD_REDUCE_SCATTER;std:stringalgConfig="ReduceScatter=level0:fullmesh";uint32_treduceType=HCCL_REDUCE_SUM;AscendC:Mc2CcTilingConfigmc2CcTilingConfig(groupName,opType,algConfig,reduceType);
1. 通过配置接口设置通信信息（可选）。12mc2CcTilingConfig.SetSkipLocalRankCopy(0);mc2CcTilingConfig.SetSkipBufferWindowCopy(1);可调用的配置接口列于下表。表1Mc2CcTilingConfig类对象的配置接口列表接口功能SetOpType设置通信任务类型。SetGroupName设置通信任务所在的通信域。SetAlgConfig设置通信算法。SetReduceType设置Reduce操作类型。SetStepSize设置细粒度通信时，通信算法的步长。SetSkipLocalRankCopy设置本卡的通信算法的计算结果是否输出到recvBuf。SetSkipBufferWindowCopy设置通信算法获取输入数据的位置。SetDebugMode设置调测模式。
1. 调用GetTiling接口，获取Tiling信息。12mc2CcTilingConfig.GetTiling(tiling->mc2InitTiling);// tiling为算子组装的TilingData结构体mc2CcTilingConfig.GetTiling(tiling->reduceScatterTiling);
