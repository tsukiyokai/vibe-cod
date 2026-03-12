# GroupBarrier使用说明-GroupBarrier-Cube分组管理(ISASI)-基础API-Ascend C算子开发接口-API-CANN社区版8.5.0开发文档-昇腾社区

**页面ID:** atlasascendc_api_07_0301
**来源：** https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/API/ascendcopapi/atlasascendc_api_07_0301.html
---

# GroupBarrier使用说明

当同一个CubeResGroupHandle中的两个AIV任务之间存在依赖关系时，可以使用GroupBarrier控制同步。假设一组AIV A做完任务x以后，另外一组AIV B才可以开始后续业务，称AIV A组为Arrive组，AIV B组为Wait组。

基于GroupBarrier的组同步使用步骤如下：

1. 创建GroupBarrier。
1. 被等待的AIV调用Arrive，需要等待的AIV调用Wait。

下文仅提供示例代码片段，更多完整样例请参考GroupBarrier样例。

1. 创建GroupBarrier。123constexprint32_tARRIVE_NUM=2;// Arrive组的AIV个数constexprint32_tWAIT_NUM=6;// Wait组的AIV个数AscendC:GroupBarrier<AscendC:PipeMode:MTE3_MODE>barA(workspace,ARRIVE_NUM,WAIT_NUM);// 创建GroupBarrier，用户自行管理并对这部分workspace清零
1. 被等待的AIV调用Arrive，需要等待的AIV调用Wait。12345678autoid=AscendC:GetBlockIdx();if(id>0&&id<ARRIVE_NUM){//各种Vector计算逻辑，用户自行实现barA.Arrive(id);}else(id>=ARRIVE_NUM&&id<ARRIVE_NUM+WAIT_NUM){barA.Wait(id-ARRIVE_NUM);// 各种Vector计算逻辑，用户自行实现}
