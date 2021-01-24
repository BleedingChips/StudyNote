基于4.25
---

**录制开始**


* UGameInstance::StartRecordingReplay

    调用该函数开始进行录制，在这之前处理一些不能录制的情况，比如 -noreplay 选项，GetWorld() 为 Null ，已经在录制等，这时候报Log进行返回。
    如果此时 UWrold::DemoNetDriver 没有被设置，会重新创建一个并设置。
       
    * 这个时候，会从配置里找与 NAME_DemoNetDriver 所绑定的 NetDriver 。其默认是空的，需要在 `Config/DefaultEngine.ini` 里边加上下面这些命令
    
        ```
        [/Script/Engine.GameEngine] 
        +NetDriverDefinitions=(DefName="DemoNetDriver",DriverClassName="/Script/Engine.DemoNetDriver",DriverClassNameFallback="/Script/Engine.DemoNetDriver")
        ```

        其中 DriverClassName 是要创建的 DemoNetDriver 的类型命，FallBack 是前面创建失败时继续创建的。如果需要用自定义的，需要继承UDemoNetDriver并把类型名覆盖上面的/Script/Engine.DemoNetDriver。

    调用 UWrold::DemoNetDriver 的 UNetDriver::SetWorld 函数绑定 UWorld ，并且从 World 里边找到类型为 ELevelCollectionType::DynamicSourceLevels 的 FLevelCollection ，并将调用 FLevelCollection::SetDemoNetDriver 函数绑定生成好的 UWrold::DemoNetDriver 。

    如果在上面新建了一个 DemoNetDriver ，则需要调用一次 UDemoNetDriver::InitListen ，在 StartRecordingReplay 中传入的一些参数会以字符串的形式直接传递到这个函数中，并初始化链接。

    * 在调用 UDemoNetDriver::InitListen 时，会首先调用 UDemoNetDriver::InitBase 。
      
      * 首先调用 UNetDriver::InitBase ，具体流程的分析下次一定。
      * 在这里会创建并重置一些属性，并且通过参数 ReplayStreamerOverride= 来创建并设置 INetWorkReplayStreamer 流推送器，和 ReplayStreamerDemoPath= 指定流送器的保存路径。

    * 新建一个新的 UDemoNetConnection 链接，并初始化。
    * 配置 LevelPrefixOverride= 相关，好像是可以捕获特定的 Level 的数据。总之以特定的选项 ELevelCollectionType::DynamicSourceLevels 和 ELevelCollectionType::DynamicDuplicatedLevels 查找 FLevelCollection ，并通过 FLevelCollection::SetDemoNetDriver 绑定 DemoNetDriver 和 FLevelCollection。
    * 对 UDemoNetDriver::ReplayStreamer 调用 INetworkReplayStreamer::StartStreaming 序列化头部。
      * INetworkReplayStreamer::StartStreaming 是一个异步操作，在这里边会创建一个 UDemoNetDriver::ReplayStreamingReady 的回调（这个回调会在其他线程调用）。
      * UDemoNetDriver::ReplayStreamingReady 这个回调会将 UDemoNetDriver::bIsWaitingForStream 这个标志位置 true 。

    否则，则调用一次 UDemoNetDriver::ContinueListen ，以恢复监听，同时参数也会以字符串的形式传进来。

    * 在里边会新建一个新的 DemoRecSpectator ，是一种特殊的 PlayerController ，这个会从 GameModeBase::ReplaySpectatorPlayerControllerClass 获取。并且把 PlayerController 的NetDriverName 设置成 UDemoNetDriver::NetDriverName 。
    
    至此，录制开始。

**录制中**

  

**录制结束**

**回放开始**

* UGameInstance::PlayReplay

  * 调用 UWrodl::DestoryDemoNetDriver 清除当前与 CurrentWorld 绑定的 UDemoNetDriver 。
  * 用相关参数构建 FURL 。
  * 用 NAME_DemoNetDriver 创建一个新的 NetDriver ，绑定 CurrentWorld::DemoNetDriver ，并调用 UDemoNetDriver::SetWorld 与 CurrentWorld 相关联。
  * 调用 UDemoNetDriver::InitConnect 。（注意，录制的时候调用的是 UDemoNetDriver::InitListen）
    * 大部分流程与录制开始时的初始化方式一致，不同的是一下几点。
      * 在调用 INetworkReplayStreamer::StartStreaming 时，所用的参数 FStartStreamingParameters Params 的成员变量 FStartStreamingParameters::bRecord 为 false ，即是启用回放模式。

**回放中**

* UDemoNetDriver::TickDispatch 
  * 需要调用 UDemoNetDriver::IsPlaying() 以及标记位 UDemoNetDriver::bIsWaitingForStream 为 True。所以这个应该是回放的时候调用的。
  * 调用 INetworkReplayStreamer::GetStreamingArchive() 并查看流送器是否准备完毕，并且流关卡是否加载完成。
  * 设置 UDemoNetDriver::SpectatorControllers 里所有 Controller 的间隔时间。
  * 调用 UDemoNetDriver::TickDemoPlayback 。
    * 调用 UDemoNetDriver::PrepFastForwardLevels 
    * 调用 UDemoNetDriver::ProcessReplayTasks 
      * 这个函数主要是在查询当前是否在执行 ReplayTask ，主要用在跳时间。
    * 从 ReplayStreamer::GetStreamingArchive 那里获取 Archive，然后查询是否播放完毕。
    * 同步时间
    * 查看是否需要加载资源，如果是的话，暂停 Channel 。
    * 循环加载数据。
    * 循环同步数据。
    * 完成快速前进。
  * 显示渐入时间？

**回放跳播**

* UDemoNetDriver::GotoTimeInSeconds
  * 查看当前有没有正在跳时间，如果没有则以 false 为参数调用输入的 FOnGotoTimeDelegate 回调并返回。
  * 否则，则新建一个 FGotoTimeInSecondsTask 。
    * 
  * 将 FGotoTimeInSecondsTask 以参数调用 UDemoNetDriver::AddReplayTask 。
    * 将 FGotoTimeInSecondsTask 存入 UDemoNetDriver::QueuedReplayTasks 中。
    * 调用 UDemoNetDriver::ProcessReplayTasks 。
      * 如果 UDemoNetDriver::ActiveReplayTask 为空，即有其他正在执行，找 UDemoNetDriver::QueuedReplayTasks 的第一个 FQueuedReplayTask 赋值到 UDemoNetDriver::ActiveReplayTask 即当前，然后调用 FQueuedReplayTask::StartTask 。
        * 派生到 FGotoTimeInSecondsTask::StartTask 
        * 记录当前的回放时间和需要跳转到的回放时间。
        * 检查当前 UDemoNetDriver 是否有 DeltaCheckpoint 。
        * 生成一个 FGotoTimeInSecondsTask::CheckpointReady 的回调，然后调用 INetworkReplayStreamer::GotoTimeInMS 。
          * 以 FLocalFileNetworkReplayStreamer::GotoTimeInMS 为例 
          * 查看是否有当前调用的 FileRequest ，如果是，则以 FGotoResult 的默认值调用 FGotoTimeInSecondsTask::CheckpointReady 。
          * 如果没有，则在 FLocalFileNetworkReplayStreamer::CurrentReplayInfo 里边的 Checkpoints 找到最近的一个 Checkpoints 的 Index ，如果找不到则用 INDEX_NONE 作为 Index ，然后用这个 Index 调用 INetworkReplayStreamer::GotoCheckpointIndex 。
            * 如果找不到对应的 CheckPoint 则重置整个流，并返回。
            * 如果找的到，并且 CheckPoint 的类型是 EReplayCheckpointType::Delta ，则重新打开文件，遍历所有的 CheckPoint ，合并数据，然后调用回调并返回。
            * 如果找的到，并且 CheckPoint 的类型是 EReplayCheckpointType::Full ，找与目标时间最近的CheckPoint ，然后调用回调并返回。
          * 在 FGotoTimeInSecondsTask::CheckpointReady 中，如果完成读取，则将结果写入 FGotoTimeInSecondsTask::GotoResult 中，否则，则还原 Device 的 DemoCurrentTime ，并且以 false 调用 UDemoNetDriver::NotifyGotoTimeFinished 。
          * 
        * 暂停 channel 。
      * 如果 UDemoNetDriver::ActiveReplayTask 有效，则调用 FQueuedReplayTask::Tick ，如果返回为 false ，则调用 FQueuedReplayTask::ShouldPausePlayback 并返回。否则，则清空 UDemoNetDriver::ActiveReplayTask 并且返回 false 。
        * 以 FGotoTimeInSecondsTask::Tick 为例。
        * 一直轮询 FGotoTimeInSecondsTask::GotoResult 是否有效，若有效，则调用 FPendingTaskHelper::LoadCheckpoint 。
          * 调用 UDemoNetDriver::LoadCheckpoint ，重置Actor的主要逻辑。

      * UDemoNetDriver::SkipTime

**回放完成**


Match State Changed from