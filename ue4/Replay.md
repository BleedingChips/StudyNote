## 元宇宙人工智能行车记录仪简析—回放系统入门

1. 简述
回放系统指通过记录游戏中产生的网络数据，并通过该记录还原游戏情景的系统。UE4自带了一套回放系统，而在项目中也是依托于这套系统来构建功能。

    若想体验UE4自带的回放系统，可以参考官方文档。
    
    回放系统的功能非常依赖于Server-Client模型下的网络同步功能，它非常类似于一个沉默的玩家加入到一个多人游戏中 ，这意味着如果一个游戏不是一个多人游戏，那么回放系统将不会完整的记录整个游戏过程（比如当你在游戏中的服务器上创建了一个Actor，但是该Actor被设置成不同步的，那么在回放中你也将不会看到这个）。而UE4本身也带了一套针对回放的同步机制，用以帮助处理时序问题。

2. 基本原理
    
    1. UE4同步原理
        
        在多人游戏中，既上面所说的Server-Client模式下，我们必须保证连接同一个服务器的各个客户端，和其所连接的服务器，其数据在任意时刻上均保持一致，而这正是网络同步的作用。

        由于现实中数据量巨大，并且存在网络延迟，所以一般的只会同步一些比较核心的数据，其余部分由客户端自行模拟。

        在UE4中，一般而言有两种同步方式：

        1. RPC（远程过程调用），指一个终端通过网络往另一个终端发送一段数据，并且在另一个终端中发起一个函数调用的行为。

            在UE4中，客户端只能发送RPC到服务器（单对单），服务器能够发送RPC到特定某个客户端（单对单），或者所有客户端（单对多）。

            一般而言，RPC的发送必须要依赖于一个Actor。同时，在RPC的两端，都必须有一个Actor的实体。

            对于单对单模式下RPC调用，则只有Owner是PlayerController的Actor才有资格调用。

            RPC的发送具有即时性，既只有当前连接的双端能接收到。

        2. 属性同步

            指一个服务器的Actor与其子Object，通过网络数据，在客户端中被创建或者更新的过程，一般而言，该功能只会单向由服务端发往客户端。

            对于一个Actor及其子Object，需要其本身被标记成可同步，并且需要通过相关性(AActor::IsNetRelevantFor)来判断某个Actor是否可被同步到某个客户端。

            对于Actor及其子Object内部的属性，可以通过特定的标记(AActor::GetLifetimeReplicatedProps)来控制其同步的方向，比如某个属性只会同步给回放，而某个属性不同步给回放。

            在UE4中，为了节省网络宽带，属性同步使用的是增量同步的形式，既服务器会按照某个特定的周期轮询所有被标记的属性，只有当某个被标记的属性在服务器上被改变时，才会在下一个检测周期中发往客户端。

            属性同步是一种增量同步，而增量同步能保持一致的前提是在执行增量同步前，两端的数据必须保持一致。所以在客户端第一次连接服务器时，必须执行一次全量同步。

            所以可以推导出一个公式：

            ```text
            当前帧服务器状态 = 当前帧新连接客户端的全量同步 = 老客户端的状态 + 当前帧的增量同步
            ```

            把公式转换可得：

             ```text
            当前状态T1 = 全量同步T1 = 全量同步T2+ ∫（T2到T1增量同步） (T1>T2)
            ```


    2. UE4回放录制/播放原理
        
        UE4的回放，采用的方式为缓存网络同步数据，并根据同步数据，在任意时间和地点还原场景。
        
        由上述公式可得，只要能获取到T1的全量同步，并且获取到从T1到T1之后的任意时间点的增量同步，即可根据公式，还原出当时的状态。
        
        所以，在任意一个时间点播放回放时，需要执行的操作如下：

        * 定位当前时间T2。
        * 找到当前时间之前的最近的一个全量同步，时间为T1。
        * 执行一次全量更新。
        * 找到T1到T2之间的所有增量同步数据（此时应该忽略掉RPC调用）。
        * 快速执行上述增量同步。
        * 从T2开始，正常根据每帧执行增量同步（之后的RPC调用需要执行）。

        理论上，从某个时间点播放回放的逻辑与客户端新连接的表现应该一致。

        由播放操作可以推断，在录制时，需要每隔一段时间缓存一次全量同步数据，并且除此之外，每隔一段时间缓存一次增量同步数据和RPC。

        在录制时， UE4 将会为录制单独创建一个专有的连接，该连接专门负责将缓存数据缓存而不是通过网络发送出去。

3. 回放系统的关键类型
    
    回放系统中有几个重要的类型：

    * `UDemoNetConnection`

        UNetContection 的子类，录制是创建的专有连接，负责模拟客户端，并将同步数据截取下来以供后续使用。

    * `UDemoNetDriver`

        UNetDriver的子类，控制回放系统的录制和播放，其复写了UNetDriver的一些方法用以控制其同步方法和数据流向。

        如果需要控制回放中的进度，播放速度等，必须通过 UDemoNetDriver 的接口来进行控制，并且UDemoNetDriver也同时控制着全量同步和增量同步的实机与实现方式。

    *  `INetworkReplayStreamer`

        控制回放系统的数据传输接口，被UDemoNetDriver创建并使用。其内置派生类有下面四种：

        * `FInMemoryNetWorkReplayStreamer`

            将回放数据保存在内存中/从内存中读取回放数据

        * `FHttpNetWorkReplayStreamer`

            将回放数据通过HTTP协议发送至其他服务器/从其他服务器通过HTTP协议拉取回放数据

        * `FLocalFileNetworkReplayStreamer`

            将回放数据保存到本地文件/从本地文件中读取回放数据

        * `FNullNetworkReplayStreamer`

            同 FLocalFileNetworkReplayStreamer ，但 FLocalFileNetworkReplayStreamer 会将数据保存到一个文件中，而 FNullNetworkReplayStreamer 是多个分散的文件保存和读取。

        一般的，我们只需要指定INetworkReplayStreamer的类型控制数据源，然后通过 UDemoNetDriver 的控制回放逻辑就能做大部分的功能。

4. 回放系统录制时产生的数据类型

    所有在回放系统录制时产生的数据均被按类型打包成一整块的数据。

    * `Mate`

        元数据，位于开头，定义了录制版本，引擎版本，录制时间等一些信息。

    * `Head`

        文件头，一般只有一个，定义了该次录制的开始关卡，和各数据的类型偏移和大小等。

    * `Event`

        一块自定义的字节块，由开发者自由控制写入和读取，一般用来传递一些全局的信息。可以在录制的结尾写入一些对局的结果信息，并且在播放之前读取。

    * `CheckPoint`

        一种特殊的Event，表示某个时刻的全量同步数据。

    * `ReplayData`

        在两个相邻时间点的CheckPoint之间所有的增量同步数据，其内部是以一帧一帧分割的小数据块，同时记录着该块数据所代表的时间。一般的，根据不同的Streamer，可以分成一个或多个数据块来进行储存。（比如HTTPStreamer会将10s作为一个分界线。如果ChunkPoint相邻的时间为15s，那么就会分为一个10s的ReplayDat和一个5s的ReplayData）。

5. 回放系统录制流程

    要标记一个回放文件，需要一个 RecordName （回放的名字），一个 FriendlyName （备注），以及指定一个处理回放数据的 ReplayStreamer 的类型。

    由于一些 ReplayStreamer 需要将 RecordName 作为一些命令的一部分，所以 RecordName 需要尽量设置成不包含特殊符号的样式。一些描述性的信息可以写在 FriendlyName 上。

    1. 开始一个录制
        
        开始一个录制时，其流程为：

        * 调用 UGameInstance::StartRecordingReplay 开启录制。
        * UGameInstance 根据配置，创建对应类型的 UDemoNetDriver。
        * UDemoNetDriver 调用 UDemoNetDriver::InitListen 开启监听。
        * UDemoNetDriver 调用 UDemoNetDriver::InitBase 初始化。该函数在录制和回放时均会被调用。
        * UDemoNetDriver 调用 UNetDriver::InitBase 初始化自身。
        * UDemoNetDriver 初始化环境，并根据环境设置，创建并初始化一个 ReplayStreamer 。
        * ReplayStreamer 的类型来源于在调用 StartRecordingReplay 时传入的参数（ ReplayStreamerOverride=XXX ）所构建的，默认值为 LocalFileNetworkReplayStreaming ，既为针对本地文件的流送器。
        * UDemoNetDriver 创建一个 UDemoNetConnection 。与普通的 Conection 不同， UDemoNetConnection 会将所有发送的数据，打包成 FQueuedDemoPacket 然后缓存到自身，以供后续UDemoNetDriver操作。

        * UDemoNetDriver 调用 UNetDriver::AddClientConnection ，加入到连接中。

        * UDemoNetDriver 收集当前环境，并对创建出来的 ReplayStreamer 调用 INetworkReplayStreamer::StartStreaming 。

        * ReplayStreamer 初始化录制环境，并产生和传输 Mate 数据。该步骤为给 ReplayStreamer 一个 UDemoNetDriver::ReplayStreamingReady 的委任，因为是录制，该委任只会在 ReplayStreamer 初始化失败的时候中止录制，否则什么都不干。

        * UDemoNetDriver 将当前关卡的路径和录制时间，写入 UDemoNetDriver::LevelNamesAndTimes 。这一般用来在多持久关卡的环境下（既游戏录制过程中，需要多次切换主关卡）的播放。

        * UDemoNetDriver 调用 UDemoNetDriver::WriteNetworkDemoHeader 写入 Head 数据。

        * UDemoNetDriver 调用 UDemoNetDriver::SpawnDemoRecSpectator 创建一个 Spectator 。在默认形况下，该 Spectator 的类型来自AGameModebase::ReplaySpectatorPlayerControllerClass 。

    2. 录制中缓存 ReplayData （增量更新数据和RPC）
        
        当缓存 ReplayData 时：

        * UWorld 调用 UWorld::BroadcastTickFlush
        * UDemoNetDriver 调用 UDemoNetDriver::TickFlush
        * UDemoNetDriver 调用 UNetDriver::TickFlush 。

            在原始代码中这里需要调用 ServerReplicateActors ，但在 UDemoNetDriver 中不会调用。真正的 ServerReplicateActors 在后续由 UDemoNetDriver 手动执行。

        * UDemoNetDriver 调用 UDemoNetDriver::DemoRecord

        * UDemoNetDriver 更新 DemoTime 。
        * UDemoNetDriver 内的 ReplayStreamer 同步更新 DemoTime 。
        * UDemoNetDriver 调用 UDemoNetDriver::TickDemoRecordFrame
        * UDemoNetDriver 调用 UDemoNetDriver::ClientConnections[0] 的 UDemoNetConnection::FlushNet。

        * 这里会将 UNetConnection 内数据按包的形式，存入到 UDemoNetConnection 中。

        * UDemoNetDriver 统计所有需要同步的Actor，并缓存到 UDemoNetDriver::PrioritizedActors 中。

            这里面会根据 Actor 本身的相关性，更新频率，同步裁剪距离等因素，筛选出当前帧所需要同步的 Actor 。

        * 对于一些与回放无相关性的 Actor ，可以通过指令 demo.RecordHzWhenNotRelevant XX 来在接下来的一段时间内跳过同步。

        * 这里可以通过 UDemoNetDriver::SetMaxDesiredRecordTimeMS 和使用指令 demo.MaximumRepPrioritizePercent 来规定最大的筛选时间，其最大筛选时间为两者之积。

            对于一般的 Actor ，对回放来说都是 ENetRole::ROLE_SimulatedProxy 。

        * UDemoNetDriver 对 UDemoNetDriver::PrioritizedActors 内的 Actor 进行优先级排序，用来控制其同步数据在每一帧的数据包中的位置（默认关闭）。

        * UDemoNetDriver 对 UDemoNetDriver::PrioritizedActors 调用 UDemoNetDriver::ReplicatePrioritizedActors 开始同步所有 Actor 。

        * UDemoNetDriver 调用 UDemoNetDriver::ClientConnections[0] 的 UDemoNetConnection::FlushNet。

        * UDemoNetDriver 调用 UDemoNetDriver::WriteDemoFrameFromQueuedDemoPackets 将 UDemoNetConnection 内的所有数据包写入到 ReplayStreamer 中。

        * 查看是否需要保存 CheckPoint ，如果需要，则进入到 CheckPoint 阶段。
        * UDemoNetDriver 更新更新计数，以控制同步频率。

        * 在调用 UDemoNetDriver::ReplicatePrioritizedActors 开始同步Actor时，其流程为下：

            * 按顺序依次对传入的 PrioritizedActors 调用 UDemoNetDriver::ReplicatePrioritizedActor ，若发生一次同步失败，则直接中止。

            * 对传入的 Actor ，若需要随网络删除，则发送 Destroy 消息。

            * 否则继续执行相关的同步
            * 对传入的 Actor ，计算其同步频率，并计算下一次的更新时间。

                Actor 的同步频率主要由其本身的 Actor::NetUpdateFrequency 控制。但在回放系统，可以通过指令 demo.MinRecordHz 和 demo.RecordHz 来规定频率的上下限。

                * 调用 Actor::CallPreReplication

            * 调用 UDemoNetDriver::DemoReplicateActor

            * 调用 Actor 所绑定的 ActorChannel 的 UActorChannel::ReplicateActor，执行正式的同步。
            * 调用 UDemoNetDriver::UpdateExternalDataForActor

            * 这个函数主要处理 FRepChangedPropertyTracker::ExternalData 内的数据。但粗略看了下，这个数据并没有被使用。

            * 检查当前同步的总用时，若超时，则直接返回失败，等待下一次Tick继续进行同步。
    
    3. 录制中缓存 CheckPoint （全量更新数据）
        缓存 CheckPoint 与缓存 ReplayData 相互独立。

        缓存 CheckPoint （全量同步）的开始位于 UDemoNetDriver::DemoRecord 函数内，内部会调用 UDemoNetDriver::ShouldSaveCheckpoint 来判断是否保存CheckPoint。该函数会根据当前时间和上一次保存CheckPoint的时间的差是否大于某个特定的值，来返回是否需要保存CheckPoint，该特定值可通过 demo.CheckpointUploadDelayInSeconds 来控制。

        * 当 UDemoNetDriver::ShouldSaveCheckpoint 返回真时，既开始调用 UDemoNetDriver::SaveCheckPoint 初始化环境。

        * UDemoNetDriver::SaveCheckpoint
        * 将所有需要网络同步的 Actor ，根据条件筛选到 UDemoNetDriver::CheckpointSaveContext::PendingCheckpointActors 里。

        * 初始化其余 UDemoNetDriver::CheckpointSaveContext 的属性。

        * 判断每次全量同步是否有时间上限，若有，则直接调用 UDemoNetDriver::TickCheckpoint 进行全量同步。该时间上限可由指令 demo.CheckpointSaveMaxMSPerFrameOverride 控制。

        * 实际上的全量同步发生在 UDemoNetDriver::TickCheckpoint 内。若存在时间上限，则由 UDemoNetDriver::SaveCheckpoint 调用，否则，则由 UDemoNetDriver::TickDemoRecord 调用。

        * 在存在全量同步时间上限的情况下， UDemoNetDriver::TickCheckpoint 内部的具体行为由 CheckpointSaveContext.CheckpointSaveState 的状态控制，其状态分别如下：

            * ECheckpointSaveState_Idle

                默认状态，该状态表示此时并不在保存 CheckPoint 的状态。

            * ECheckpointSaveState_ProcessCheckpointActors

                由 UDemoNetDriver::SaveCheckPoint 设置，初始状态，表示要开始进行对单个 Actor 的全量同步。

                其流程如下：

                * 将 UDemoNetConnection 的 UNetConnection::ResendAllDataState 设置成 EResendAllDataState::SinceOpen 。

                * 对所有在 UDemoNetDriver::CheckpointSaveContext 储存的 Actor ，调用 UDemoNetDriver::ReplicateCheckpointActor 。

                * 调用 AActor::CallPreReplication

                * 调用 UDemoNetConnection::DemoReplicateActor 执行属性同步。

                * EResendAllDataState::SinceOpen 能保证该次属性同步是全量同步。

                * 调用 UDemoNetDriver::UpdateExternalDataForActor

                * 查询剩余时间，若消耗的时间大于最大同步上限，返回否，否则返回真。

                * 刷新其所含有的 UDemoNetConnection 的数据。

                * 如果全部 Actor 均被同步完成，则状态切入成 ECheckpointSaveState_SerializeDeletedStartupActors。
                * 否则，则在下一帧中继续同步。
            
            * ECheckpointSaveState_SerializeDeletedStartupActors

                将所有需要销毁的 Actor 的相关信息写入数据。写入完成后，进入 ECheckpointSaveState_CacheNetGuids 阶段。

            * ECheckpointSaveState_CacheNetGuids

                调用 UDemoNetDriver::CacheNetGuids ，将 UNetDriver 内部的 GuidCache::ObjectLookup 的所有 Guids 缓存到 UDemoNetDriver::NetGuidCacheSnapshot 上。

                切入到 ECheckpointSaveState_SerializeGuidCache 。

            * ECheckpointSaveState_SerializeGuidCache

                调用 UDemoNetDriver::SerializeGuidCache ，将 Guids 和对应的信息（指若该Guids是一个静态Guids，或者该Guids能通过名字找到对应的唯一值，那么就需要把路径或者名字一起写入）一起写入CheckPoint数据。

                该步骤可能会被分帧，若执行完毕，则将状态切入到 ECheckpointSaveState_SerializeNetFieldExportGroupMap 。

            * ECheckpointSaveState_SerializeNetFieldExportGroupMap

                *  对 UDemoNetConnection 内绑定的 PackageMap 调用 UPackageMapClient::SerializeNetFieldExportGroupMap 将数据写入 CheckPoint 。

                * 将状态且入 ECheckpointSaveState_SerializeDemoFrameFromQueuedDemoPackets 。

            * ECheckpointSaveState_SerializeDemoFrameFromQueuedDemoPackets

                调用 UDemoNetDriver::WriteDemoFrameFromQueuedDemoPackets ，将当前帧的地图信息，UDemoNetConnection 内所保存的所有同步数据一同写入 CheckPoint 。

                执行完毕后，状态切入 ECheckpointSaveState_Finalize 。

            * ECheckpointSaveState_Finalize

                告知 ReplayStreamer 所有数据已完毕，可以准备流送，同时将状态切入 ECheckpointSaveState_Idle ，完成所有 CheckPoint 的保存。

    4. 录制中缓存 Event
        
        在录制的过程中，可以根据需要写入各种自定义的 Event ，该 Event 用以给开发者储存一些额外的全局数据。该类型的数据打包或者转存，均由 ReplayStreamer 完成，UDemoNetDriver 只提供必要信息和作为一个接口。

        为了标记一个 Event ，需要提供以下几个信息：

        * FString Name 必须唯一，重复保存会使得后续的 Event 覆盖掉前面的 Event 。若不提供，可由 ReplayStreamer 自动生成。
        * FString GroupName 表示当前 Event 所属的类型，后续可以通过特定的方式筛选出所有该类型的 Event 。
        * FString MateData 表示当前 Event 的描述性符号，用以提供筛选。
        * uint32 TimeInMS 给 Event 标记一个时间，一般情况下时储存当前时间的 DemoTime 。
        * TArray<uint8> Datas Event 的所有数据。
            一般情况下，在录制的时候可以使用相关的接口来储存 Event

            * UDemoNetDriver::AddEvent

                该接口需要提供 GroupName ， Mate ， 和 Datas 。其余的 Name 由 ReplayStreamer 自动生成， TimeInMS 则为当前的时间。

            * UDemoNetDriver::AddOrUpdateEvent

                该接口比上述接口多接收了一个 Name 的参数。

            另：部分类型的 ReplayStreamer 不支持 Event 。

    5. 结束录制

        若需要结束录制，其流程为如下：

        * 调用 UWorld::StopRecordingReplay

            该函数同时实现两个功能，既停止录像和停止播放。

        * 调用 UWrold::DestroyDemoNetDriver
        * 调用 UDemoNetDriver::StopDemo
        * 调用 UDemoNetDriver::OnDemoFinishRecordingDelegate 回调。
        * 对 ClientConnections[0] 调用 UDemoNetConnection::Close 关闭连接。
        * 对 ReplayStreamer 调用 INetworkReplayStreamer::StopStreaming 关闭流送器。
        * 重置录制状态。
        * 调用 UEngine::DestroyNamedNetDriver 删除 UDemoNetDriver。
        * 重置 UWorld::DemoNetDriver
        * UDemoNetDriver 内还有关 DetalCheckPoint 和 LevelStreaming 的相关处理，还有暂停/继续录制等操作。由于篇幅有限，暂不在此多做介绍。

6. 回放系统播放流程

    1. Event 的拉取
    
        Event 与播放相互独立，可以在未播放或者播放中进行拉取。一般情况下，Event 的拉取，都是由用户发出请求，并提供一个委任，然后 ReplayStreamer 会异步调用委任返回请求结果。

        拉取 Event 的方式有很多：

        * 根据 Name 拉取。

            效率最高的的方式，但你需要提供所需要的 Event 的 Name ，并且相对应的，只能拉取一个。

            相关接口：

            * UDemoNetDriver::RequestEventData
            * INetworkReplayStreamer::RequestEventData
                
                上述两个接口的区别在于， UDemoNetDriver 必须在播放状态下， INetworkReplayStreamer 则必须由自己创建对应类型的 Streamer ，但可以有相关重载，能根据对应的文件名，在非播放状态下获取。

        * 根据 GroupName 拉取所有相关的 Event 。

            可以拉取所有具有所要求 GroupName 的所有 Event 。根据 Event 之间的存放距离，可能会有多次的IO。

            * UDemoNetDriver::RequestEventGroupDataForActiveReplay
            * INetworkReplayStreamer::RequestEventGroupData

        * 根据 GroupName 筛选出对应的 Event 信息。

            这种方式，只能拉取到 Event 的 Name ， Mate ， Time 等数据，拿不到其对应的 Datas 。一般情况下，需要通过获取到的 Name ，重新通过上述的方式1拉取 Datas 。这里拉取 N 次 Event 需要 N+1 次异步操作，但好处在于可以拉取相同 GroupName 中的特定某几个数据。

            * UDemoNetDriver::EnumerateEvents
            * INetworkReplayStreamer::EnumerateEvents
                
                Event 的存储完全取决于 ReplayStreamer 的实现，所以需要假设同样 GroupName 的 Event 在储存的物理位置上可能相距较远。

                考虑到性能， Event 的储存方式和拉取方式需要根据 Event 大小，数量以及使用方式来特异化处理。

    2. 回放的播放

        回放的播放支持在被录制的游戏已经结束之后播放，或者游戏中边录制边播放（有延时），获取是否是正在录制的回放可以用 INetworkReplayStreamer::IsLive() 函数获得。

        对于正在录制的回放（IsLive() == true），有些功能可能不支持。

        在回放中， 一般 ReplicatedActor 的 GetNetMode() 的返回值为 NM_Client ，GetLocalRole() == ENetRole::ROLE_SimulatedProxy 。因为这时候连接的是一个 DemoNetDriver 模拟的假DS 。

        1. 开始播放
        
            要播回放，需要根据回放文件名，和一些控制播放的选项，调用 UWorld::PlayReplay 开始进行播放。

            * 调用 UWorl::PlayReplay
            * 根据配置，创建 UDemoNetDriver 。

            * 对 UDemoNetDriver 调用 UDemoNetDriver::InitConnect 。

            * 调用 UDemoNetDriver::InitBase 创建 ReplayStreamer 并初始化相关设置。
            * 创建 UNetConnection 作为模拟服务器的连接，并将其缓存至 UDemoNetDriver::ServerConnection 。
            * 对 UNetConnection 调用 UNetConnection::InitConnection ，初始化连接。
            * 初始化相关设置。
            * 告知 ReplayStreamer 开始流送。

                该步骤会给与 ReplayStreamer 一个委任，既当 ReplayStreamer 初始化完毕后，会异步调用 UDemoNetDriver::ReplayStreamingReady 。

                与此同时，虽然 ReplayStreamer 尚未准备完毕，Tick 函数也会一直被调用。

            * UDemoNetDriver::TickDispatch

                该函数专门处理，但由于 UDemoNetDriver::ReplayStreamingReady 未被调用，所以一直在空转，只调用 UNetDriver::TickDispatch 。

                等 ReplayStreamer 准备完毕，会调用 UDemoNetDriver::ReplayStreamingReady 。

            * UDemoNetDriver::ReplayStreamingReady
                
                若 ReplayStreamer 初始化失败则直接停止录制。

            * 调用 UDemoNetDriver::InitConnectInternal ，进行环境的初始化。

            * 调用 UDemoNetDriver::ResetDemoState 重置环境属性。

            * 调用 UDemoNetDriver::ReadPlaybackDemoHeader 从 ReplayStreamer 读取 Head 数据，并初始化相关设置。若失败，则直接返回初始化失败。

            * 调用 UDemoNetDriver::CreateInitialClientChannels ，创建 ControlChannel 。

            * 根据 Head 数据（内含录制时的关卡信息），打开对应关卡。

            * 该步骤默认是立即返回的（既直接调用 OpenLevel 的方式打开），但可以通过指令 demo.AsyncLoadWorld 1，或者在播放的时候传入 AsyncLoadWorldOverride=1 来进行设置，使其进行异步加载（既用 SeamLessTravel 的方式打开关卡）。

            * 此时会给与一个异步的委任 UDemoNetDriver::OnPostLoadMapWithWorld ，在加载完关卡后处理，该委任会重置跟关卡相关的所有信息。

            * 创建新的 UDemoPendingNetGame ，作为 FWorldContext::PendingNetGame 的值。

            * 返回成功。

            * 判断设置，是否需要自动跳转到回放的结尾。

                该功能可通过指令 demo.JumpToEndOfLiveReplay 开启或关闭（默认开启）。若不自动跳转，则默认在起始点。

                如果是 IsLive 并且当前录制进度大于 15s 。
                如果初始化耗时少于 10ms，调用 UDemoNetDriver::JumpToEndOfLiveReplay 直接跳转至末尾附近的位置。
                
                否则

                启用一个异步 Task 去调用 UDemoNetDriver::JumpToEndOfLiveReplay 。

            * 调用委任 UDemoNetDriver::OnDemoStarted 。

            * 于此同时， UDemoNetDriver::TickDispatch 会被调用，但同时的，也在空转，直到所有的 LevelStreaming 加载完毕。

        2. 正常播放

            正常播放时由 UDemoNetDriver::TickDispatch 驱动。

            * UDemoNetDriver::TickDispatch
                * 调用 UNetDriver::TickDispatch
                * 根据设置调整 AWorldSetting::DemoPlayTimeDilation ，这个值控制回放的播放速度，可由指令 demo.TimeDilation 设置。也可直接通过 AWorldSetting 设置该值，但要保证 demo.TimeDilation 的值大于 0.0f 。
                * 设置 SpectatorController 的时间膨胀，使得当暂停时， SpectatorController 依然能够正常 Tick 。
                * 设置 SpectatorController 所绑定的 SpectatorPawn 的时间膨胀。
                * 调用 UDemoNetDriver::TickDemoPlayback 。
                * 此时可以根据指令 demo.GotoTimeInSeconds 让回放跳转到对应的时间，或者使用指令 demo.SkipTime 让回放往后跳转特定的时间，该操作会让会放进入时间跳转流程（指从一个时间点直接进入另一个时间点）。

                    该流程会产生一个异步的 Task 去处理时间跳转的情况。

                * 更新当前回放的最长时间（一般针对 IsLive 的情况）。

                * 若当前需要处理时间跳转，则立即返回。

                * 理当播放完毕后的行为，比如退出，比如重头开始播等。

                * 更新当前的 DemoTime 。

                * 告知 ReplayStreamer 更新数据至 DemoTime 。

                * 若 ReplayStreamer 内的同步数据无效，则暂停所有的 Channel ，并退出该函数。

                * 若 ReplayStreamer 内的数据有效，则开启所有的 Channel 。

                * 调用 UDemoNetDriver::ConditionallyReadDemoFrameIntoPlaybackPackets ，该函数会根据 DemoTime 将 ReplayStreamer 内的 ReplayData 数据填入 UDemoNetDriver::PlaybackPackets 中。

                * 循环调用 UDemoNetDriver::ConditionallyProcessPlaybackPackets 执行增量同步。

                * 该函数每执行一次，则消耗一次单帧的 ReplayData ，但若播放时间过快，有可能在一个 Tick 的时间内需要吃掉多帧的 ReplayData 。

                * 清空已消耗的 ReplayData 数据。

        3. 时间跳转

            当需要从一个时间点直接跳转到另一个时间点时执行，该行为一般由 UDemoNetDriver::GoToTimeInSecond 触发。

            当执行时间跳转时，会使得 UDemoNetDriver::TickDispatch 执行会在中途中断。

            * UDemoNetDriver::GoToTimeInSecond

                查看当前是否有另一个时间跳转的 Task 在执行，如果是，则返回跳转失败。

                创建并加入一个 FGotoTimeInSecondsTask 。

            * FGotoTimeInSecondsTask::StartTask

            * 对 ReplayStreamer 调用 INetworkReplayStreamer::GotoTimeInMS 。并提供回调 FGotoTimeInSecondsTask::CheckpointReady 。

                该函数会让 ReplayStreamer 准备这个离时间点最近的前一个 CheckPoint 数据，以及 Checkpoint 时间点到目标时间点中间内的所有 ReplayData 数据。当准备完毕后（通常指将数据读值内存中），调用 FGotoTimeInSecondsTask::CheckpointReady 。

            * 对 UDemoNetDriver 调用 UDemoNetDriver::PauseChannels ，关闭所有的 Channel 。

            * FGotoTimeInSecondsTask::CheckpointReady

            * 将读取后的结果写入 FGotoTimeInSecondsTask::GotoResult ，若读取失败，则直接调用 UDemoNetDriver::NotifyGotoTimeFinished(false) 。

            * FGotoTimeInSecondsTask::Tick

            * 每帧调用，查询 FGotoTimeInSecondsTask::GotoResult 是否被设置，若被设置，则直接调用 FPendingTaskHelper::LoadCheckpoint 。

            * FPendingTaskHelper::LoadCheckpoint

            * 调用 UDemoNetDriver::LoadCheckpoint 。
            * 将 SpectatorController ，和其 ViewTarget 加入到 UDemoNetDriver::NonQueuedActor 列表中。

            * 关闭所有 Channel 。

            * 调用多播委任 FNetworkReplayDelegates::OnPreScrub 。

            * 遍历场景中的所有 Actor ，查询所有需要保留的 Actor 。一般依靠如下规则：

                * 跟 SpectatorController 有关，Owner 是 SpectatorController ，或者是 ViewtingTarget ViewPawn 。

                * AActor::bReplayRewindable 为真。
                * AActor::IsNetStartupActor() 返回真（一般为场景中的Actor）。其中，若该 Actor 的 AActor::bReplayRewindable 为真，还会调用 AActor::RewindForReplay 。
            * 调用 UWorld::Destory 删除剩下的所有 Actor 。

            * 停止所有粒子特效

            * 遍历所有 channel ，取消 Channel 与 Actor 的关联。

            * 清空所有旧的回放数据。

            * 调用 UDemoNetDriver::CleanUpSplitscreenConnections 清除 Channel 内的旧数据。

            * 关闭并清理 UDemoNetDriver::ServerConnection 的连接。

            * 调用一次GC

                该功能可由指令 demo.LoadCheckpointGarbageCollect 开启或关闭，默认开启。

            一般的，执行到此处时，可能会有大量的 Actor 被销毁，并且接下来还会在短时间内新创建大量的 Actor ，所以为了节省内存，一般是建议开启。

            * 创建一个 UNetConnection 的新连接，并缓存到 UDemoNetDriver::ServerConnection 中。

            * ServerConnection 调用 UNetConnection::InitConnection 开始连接。

            * 创建新的 ClientChannel ，并且与 SpectatorController 绑定。

            * 从 CheckPoint 数据内还原场景。

            * 调用 UDemoNetDriver::SkipTimeInternal 设定后续 FastForward （正常播放前的增量同步）的目标时间。

            * 执行后续的同步

            * 再执行过一次 CheckPoint 之后，还需要再下一次的 Tick 调用一次 UDemoNetDriver::TickDemoPlayback ，用以完成 FastForward 。

            * 调用 UDemoNetDriver::TickDemoPlayback 。同普通播放时的逻辑
            * 调用 UDemoNetDriver::FinalizeFastForward 一次，下一帧后不再调用，除非又读取一次 CheckPoint 。

            * 发所有 Actor 的属性同步 Notifies 。

            * 重置 FastForward 环境。

            * 调用 UDemoNetDriver::NotifyGotoTimeFinished(true) 。

            * 该函数用于通知外部调整进度完毕。

7. C1中的回放系统主要架构
由于回放系统有相同的功能点但有不同的业务模式，所以将系统拆分到如下几个功能块：

UBFGameModeRecordComponent

作为 GameMode 的一个 ActorComponent ，用来控制回放录制的开始和停止时机，并控制回放录制中的相关参数。

派生类：

UBFGameModeRecordBMComponent

爆破模式专用的录制控制器。内部会将所有的阶段所对应的 DemoTime 合并成一个独立的类型，然后在 PostGameWaiting 阶段写入回放文件中。

UBFReplayEngine

负责系统间交互，作为一个 Component 附加在 UGameInstance 上。其主要功能有：

开启和关闭录制，并维护相关状态。
创建和维护Director，并提供一些共用接口的封装，比如UBFReplayEngine::IsReplaying()可返回当前是否在播放状态。
与一些局外功能的交互封装，如LoadingUI，返回大厅等。
注册事件回调，并在事件发生的时候传入到 UBFReplayDirector 中。
派生类：

UBFReplayVehicle

项目特化的一些配置，控制精彩时刻和赛事回放，大神观战的具体 Director 类型。

UBFReplayDirector

由 UBFReplayEngine 创建，其生命周期由自己管理，负责播放回放中的流程控制，直接与 HUD 相关业务系统进行交互，主要的业务功能都在这上。

其功能有：

与 IBFReplayDriverInterface 交互，控制播放过程，如暂停，快速播放，切进度等。并将相关信息暴露给相关HUD。
与 UBFReplayEngine 交互，开启播放，控制LoadingUI的关闭，退出回放的后续处理。
与 IBFReplayPlayerControllerInterface 交互，控制镜头的切换，移动模式的切换等。
另外，UBFReplayDirector的生存周期与回放没有强相关，所以实际上可以在开启一个回放后，通过切换不同的Director来执行不同的流程。

目前项目中的派生类有：

UBFHeroShowTimeDirector 用来处理精彩时刻相关业务逻辑。
BP_Director_HeroShowTime 对应的蓝图，一般用来做配置。
UBFFreeLancerDirector 用来处理赛事回放相关业务逻辑。
BP_Directo_FreeLancer 对应的蓝图，一般用来做配置。
UBFGreatGodDirector 用来处理大神观战相关业务逻辑。
BP_Director_GreatGod 对应的蓝图，一般用来做配置。
IBFReplayDriverInterface

一个面向 UDemoNetDriver 的抽象，其提供了一些接口用来与 UDemoNetDriver 和 FBFEventWrapper 来进行交互。

一般的， UBFReplayDirector 应该只用该接口进行交互。

UBFReplayNetDriver

UDemoNetDriver 的派生类，同时继承了 IBFReplayDriverInterface ，其主要功能是为了给 UDemoNetDriver 扩展一些项目特化的功能。

IBFReplayPlayerControllerInterface

ReplayPlayerController的一个接口，作为 UBFReplayDirector 和 SpectatorController 沟通的桥梁。一般作为 3C 相关功能的接口。

FBFEventWrapper

外围设施，提供数据类型和Event之间的序列化，并提供发送读取Event的相关功能的封装。