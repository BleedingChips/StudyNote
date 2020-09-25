[官方地址](https://docs.unrealengine.com/zh-CN/Gameplay/Networking/Overview/index.html)

网络模式说明：
1. **独立(Standalone)**  游戏作为服务器运行，不接受远程客户端连接。参与游戏的玩家必须为本地玩家。此模式用于单人游戏和本地多人游戏。其将运行本地玩家适用的服务器逻辑和客户端逻辑。
2. **客户端(Client)** 游戏作为网络多人游戏会话中与服务器连接的客户端运行。其不会运行服务器逻辑。
3. **聆听服务器(ListenServer)** 游戏作为主持网络多人游戏会话的服务器运行。其接受远程客户端中的连接，且直接在服务器上拥有本地玩家。此模式通常用于临时合作和竞技多人游戏。
4. **专属服务器(Dedicvated Server)** 游戏作为主持网络多人游戏会话的服务器运行。其接受远程客户端中的连接，但无本地玩家，因此为了高效运行，其将废弃图形、音效、输入和其他面向玩家的功能。此模式常用于需要更固定、安全和大型多人功能的游戏。

网络权限：
1. **无(None)**  Actor在网络游戏中无角色，不会复制。
2. **授权(Authority)** Actor为授权状态，会将其信息复制到其他机器上的远程代理。一般在服务器上。（若为聆听服务器，那么服务器控制的和主机玩家控制的是同一个Actor，所以虽然该Actor被玩家直接控制，但也是授权模式）
3. **模拟代理(Simulated Proxy)** Actor为远程代理，由另一台机器上的授权Actor完全控制。网络游戏中如拾取物、发射物或交互对象等多数Actor将在远程客户端上显示为模拟代理，一般为服务端/其他客户端控制的Actor在本机上的投影。
4. **自主代理(Autonomous Proxy)** Actor为远程代理，能够本地执行部分功能，但会接收授权Actor中的矫正。自主代理通常为玩家直接控制的actor所保留，如pawn。

一般Actor的权限由两个ENetRole定义，分别表示当前Actor的权限(LocalRole)以及远程连接中的权限(RemoteRole)。连接权限表示该链接链接的Actor的权限。若有三个玩家，则其分别属性如下表所示（专属服务器模式下）：

|玩家|Actor A Role:RemoteRole|Actor B Role:RemoteRole|Actor C Role:RemoteRole|
|---|---|---|---|
|服务器|Authority:SimulatedProxy|Authority:SimulatedProxy|Authority : SimulatedProxy|
|A|AutonomousProxy : Authority|SimulatedProxy : Authority|SimulatedProxy : Authority|
|B|SimulatedProxy : Authority|AutonomousProxy : Authority|SimulatedProxy : Authority|
|C|SimulatedProxy : Authority|SimulatedProxy : Authority|AutonomousProxy : Authority|

获取Actor的权限所用函数如下:

```cpp
ENetRole Actor::GetLocalRole() const;
ENetRole Actor::GetRemoteRole() const;
```

客户端拥有权

    服务器上，所有玩家的Pawn都拥有一个PlayerController，而在客户端上，只有玩家所直接控制的Pawns才拥有一个PlayerController。可以用PlayerController::IsLocalController()来判断该Controller是否是由玩家直接控制的Controller。

Actor的相关性与优先级

    可以通过复写相关函数来决定Actor的更新频率和优先级。

角色移动

    角色移动应该是基于帧同步的。客户端需要纪录多个时刻的移动，然后打包发给服务器，服务器会根据记录，尝试模拟该行为。如果模拟最后的结果等于客户端的结果，那么就使用客户端的结果，并返回一个消息给客户端，并且会将该位置同步给其余客户端，若不等于，那么服务端就会将服务端本地的结果返回给客户端，而客户端会根据这个结果重置本地的结果。

属性同步

    在UPROPERTY()宏内添加 Replicated （见[1]）或 ReplicateUsing （见[2]）说明符，或者在蓝图的Detail面板中将其设置成可复制。当服务器中的Actor发生改变时，会自动同步更新客户端中的变量。

    >PS
    1. 表示该值会参与同步。
    2. 使用方式为 在UPROPERTY(ReplicateUsing = RepFunctionName) Xxx F; UFUNCTION() void RepFunctionName();

