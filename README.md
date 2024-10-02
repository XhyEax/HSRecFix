# HSRecFix

## 声明
**仅供学习交流，禁止商业使用。转载请保留原作者信息。**

使用软件过程中，发生意外造成的损失由使用者承担。您必须在下载后的24小时内从计算机或其他各种设备中完全删除本项目所有内容。您使用或者复制了以上的任何内容，则视为已接受此声明，请仔细阅读。

## 功能
【炉石传说国服】掉线重连修复补丁，修复游戏结算（斩杀对手或对手投降）时，*掉线重连失败后游戏卡死*的问题（基于修改`Assembly-CSharp.dll`）

## 使用
打开炉石安装目录（如`D:\Program Files\Hearthstone\Hearthstone_Data\Managed`），解压并替换`Assembly-CSharp.dll`文件后，重启游戏即可。


## 分析过程
### 1. 查看日志
查看`Player.log`可知，游戏卡死是由于客户端和游戏服务端重连失败导致。
日志中提示的字符串`你的游戏与暴雪战网服务断开连接。你可以尝试再次启动游戏来重新连接。`对应`GLOBAL_ERROR_NETWORK_LOST_GAME_CONNECTION`。

```
[GameNetLogger] Network.OnGameServerConnectEvent() - Connecting to game server with error code ERROR_OK
[Hearthstone] [Asset] Updating ref count from bundle that's not currently open: essential_base_enus-content-0.unity3d|NotFound
[Asset] Updating ref count from bundle that's not currently open: essential_base_enus-content-0.unity3d|NotFound
[BattleNet] [] ClientConnection.ReceiveCallback failed. bytesReceived:0, error:Success
[GameNetLogger] Network.OnGameServerDisconnectEvent() - Disconnected from game server with error 3005 ERROR_RPC_PEER_DISCONNECTED
[GameNetLogger] ReconnectMgr.CheckGameplayReconnectRetry() - calling StartGame_Internal
[GameNetLogger] GameMgr.ReconnectGame() - GameReconnectType:GAMEPLAY
[GameNetLogger] GameMgr.ChangeFindGameState() - state: CLIENT_STARTED, previous state: SERVER_GAME_CONNECTING
[GameNetLogger] GameMgr.DoDefaultFindGameEventBehavior() -  FindGameEventData: CLIENT_STARTED
[GameNetLogger] GameMgr.ChangeFindGameState() - state: SERVER_GAME_CONNECTING, previous state: CLIENT_STARTED
[GameNetLogger] GameMgr.DoDefaultFindGameEventBehavior() -  FindGameEventData: SERVER_GAME_CONNECTING
[GameNetLogger] Network.GotoGameServe() - gameStateIsInitial False, IsRestoringGameState: False
[Hearthstone] [GameNetLogger] *** Network Game Connection State ***  
            ***************************
[Hearthstone] Error.AddDevFatal() - message=Network.GotoGameServe() - was called when we're already waiting for a game to start. gameStateIsInitial False, IsRestoringGameState: False
Error.AddDevFatal() - message=Network.GotoGameServe() - was called when we're already waiting for a game to start. gameStateIsInitial False, IsRestoringGameState: False
[Hearthstone] Error.AddDevFatal() - message=Network.GotoGameServe() - was called when we're already waiting for a game to start. gameStateIsInitial False, IsRestoringGameState: False
Error.AddDevFatal() - message=Network.GotoGameServe() - was called when we're already waiting for a game to start. gameStateIsInitial False, IsRestoringGameState: False
[GameNetLogger] GameMgr.OnServerResult() -  result: 2
[GameNetLogger] GameMgr.OnGameCanceled()
[GameNetLogger] Network.GetGameCancelInfo()
[GameNetLogger] GameMgr.ChangeFindGameState() - state: SERVER_GAME_CANCELED, previous state: SERVER_GAME_CONNECTING
[Hearthstone] [GameNetLogger] *** Network Game Connection State ***  
            ***************************
[Hearthstone] Error.AddFatal() - message=你的游戏与暴雪战网服务断开连接。你可以尝试再次启动游戏来重新连接。
Error.AddFatal() - message=你的游戏与暴雪战网服务断开连接。你可以尝试再次启动游戏来重新连接。
```
### 2. 逆向分析
使用`dnSpy`反编译`Assembly-CSharp.dll`，搜索`GLOBAL_ERROR_NETWORK_LOST_GAME_CONNECTION`，其中`ReconnectMgr.OnGameReconnectTimeout`有引用:
![](https://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/hsfix-1.png)

![](https://xhy-1252675344.cos.ap-beijing.myqcloud.com/imgs/hsfix-2.png)

分析代码可知，如果当前游戏场景为Gameplay，弹出断开连接的提示后，函数直接return。

而`AttemptToRestoreGameState`实际上会调用`gameMgr.FindGame`用现有参数连接游戏，失败则弹出`GLOBAL_RECONNECT_TIMEOUT	无法重新连接到游戏。`	

### 3. 修改逻辑
去除return，走到`AttemptToRestoreGameState`分支。

由于对局服务器实际上已关闭，故连接必然失败，从而回到主菜单，而不是游戏卡死，导致重开后卡门（排队 > 15分钟）。
