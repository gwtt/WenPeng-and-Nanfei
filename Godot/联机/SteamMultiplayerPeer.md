首先我在实验Godot联机的时候，使用了GodotSteam的extension插件形式。然后联机本质上用的还是p2p那一套。导致我联机的时候比较卡顿，也是发现了这个问题，问了AI才得知《梦之形》使用了godot的SDR技术，换言之，用了steam的中继服务器。需要下载下面版本的：
![[Pasted image 20251208170801.png]]


## 概念对比
### Manual Steam P2P

**就像“传统的邮局寄信”**。

- **发送端**：你需要拿出一张纸，把数据写上去，折好，装进信封，写上对方的 Steam ID，贴上邮票，然后交给邮局（Steam）。
    
- **接收端**：你需要一直盯着自家邮箱（_process 里的 while 循环）。如果有信来了，你要拆开信封，把纸展开，根据信的内容（Message Key）去查字典：“哦，如果写的是 rpc，我就要去调用某个函数”。
    
- **缺点**：太累了！稍微写错一个字符，信就丢了。而且如果你想同步一个不断移动的球，你得疯狂写信，手都写断了。

### SteamMultiplayerPeer

**就像“打电话 / 对讲机”**。

- **机制**：Godot 引擎其实自带了一套非常高级的对讲机系统（High-level Multiplayer API）。
    
- **作用**：SteamMultiplayerPeer 的作用就是把这台对讲机的**信号线**插到了 Steam 的网络上。
    
- **体验**：你不需要管信封、邮票、拆信这些破事。你只想让队友开枪，你就对着对讲机喊一句：“开枪！”（rpc），Godot 会自动帮你处理所有的打包、发送、接收和执行。

## 代码层面对比

### 1. 发送消息

- **旧方法**：你需要手动构建字典，转成字节流。

 ```Gdscript
    # 你的旧代码
    var data = {"message": "attack", "damage": 10}
    send_p2p_packet(target_id, data, Steam.P2P_SEND_RELIABLE)
```

- **新方法**：直接调用函数。
```Gdscript
    # 新代码
    attack.rpc(10) # 这一行代码就在所有人的电脑上执行了 attack(10)
```

### 2.接收消息

- 旧方法：由于不知道什么时候收到包，必须每帧死循环去读

```GDscript
# 你的旧代码
func _process(delta):
    var size = Steam.getAvailableP2PPacketSize(0)
    while size > 0:
        var packet = Steam.readP2PPacket(size, 0)
        # ... 然后写几十行 if/else 来判断 message 是什么
```

- 新方法：完全不需要写代码
	- 只要你的函数头顶上有 @rpc 标记，Godot 会自动在后台监听，收到数据自动触发该函数。

## 3.最佳实践

在 Godot 4 + SteamMultiplayerPeer 的架构下，我们通常把同步分为三类：
### 1. 瞬时动作 (Events) -> 使用 RPC

像“开枪”、“播放音效”、“受到伤害”、“聊天”这种由于玩家点击触发的、一次性的事件。
- **怎么写**：

```GDscript
# call_local: 我按下键，我自己也要看到开枪，同时别人也要看到
# any_peer: 允许任何玩家调用这个函数（不仅仅是房主）
# reliable: 必须送达（像开枪这种大事，不能丢包）
@rpc("call_local", "any_peer", "reliable")
func shoot():
    play_shoot_animation()
    spawn_bullet()

# 像走路时的脚步声，丢几个包也无所谓，用 unreliable 更快
@rpc("call_local", "any_peer", "unreliable")
func play_footstep():
    audio_player.play()
```

### 2.持续状态 (State/Position) -> 使用 MultiplayerSynchronizer

这是 Godot 的神器。像“玩家位置”、“血量”、“旋转角度”这种每时每刻都在变的数据，**千万不要用 RPC 去发！**（否则网络会堵死）。

- **怎么做**：
    
    1. 在玩家场景里添加一个节点：MultiplayerSynchronizer。
        
    2. 在右侧属性栏点击 Replication (复制配置)。
        
    3. 把你想要同步的属性（比如 position, rotation, health）勾选上。
        
- **为什么好用**：
    
    - 它会自动根据网络状况调整发送频率。
        
    - **最重要的是**：它只会在数据发生变化时发送，省流量。
        
    - 它可以设置 Interpolation (插值)，比如网络卡了 100ms，它会自动平滑过渡移动，而不是让角色瞬移。

### 3.动态生成物体 (Spawning) -> 使用 MultiplayerSpawner

比如“发射子弹”、“掉落装备”。以前你需要手动通知所有人“在坐标(x,y)生成一个子弹”。

- **怎么做**：
    
    1. 在场景根节点添加 MultiplayerSpawner。
        
    2. 在 Auto Spawn List 里把你的子弹预制体（Prefab）加进去。
        
    3. **代码**：只需要在**房主（Host）** 电脑上 add_child(bullet)。
        
    4. **结果**：Godot 会自动在所有客机的电脑上生成同样的子弹。

## 4.实战

当你开始写代码时，按照这个逻辑来思考：

|   |   |   |   |
|---|---|---|---|
|游戏需求|推荐方案|例子|设置建议|
|**玩家移动**|MultiplayerSynchronizer|走路、跳跃|勾选 position, velocity|
|**玩家属性**|MultiplayerSynchronizer|血量、弹药量|勾选 current_health|
|**开火/技能**|RPC|按下 Q 键放技能|@rpc("call_local", "reliable")|
|**特效/声音**|RPC|爆炸特效、脚步声|@rpc("call_local", "unreliable")|
|**生成怪物/子弹**|MultiplayerSpawner|刷怪、射击产物|房主生成，自动同步|
|**切换关卡**|RPC|所有人进入下一关|房主调用 load_level.rpc()|

## 5. 建议

1. **权限分离（Authority）**：  
这是最容易晕的地方。你需要时刻问自己：**“这段代码是谁在跑？”**

- **Master (Authority)**: 通常是房主，或者是控制这个角色的玩家。只有他能修改数据（比如扣血）。
    
- **Puppet (Remote)**: 也就是其他玩家电脑上显示的“你”。他们只能接收数据，不能修改。
    

代码里常用的判断：
```GDscript
func _physics_process(delta):
    # 只有控制这个角色的玩家才处理输入
    if not is_multiplayer_authority(): 
        return 
    
    # 处理移动逻辑...
    move_and_slide()
```

2. **别急着写大厅**

先用 Godot 提供的简单界面，把两个小人在屏幕上跑通（你可以开两个游戏窗口测试，Godot 编辑器支持同时运行多个实例）。等游戏逻辑通了，最后再接 Steam 的邀请/大厅功能。

3. **SDR 的优势**

因为你用了 SteamMultiplayerPeer，当你调用 rpc 或使用 Synchronizer 时，数据就是通过你之前问的 **SDR 架构** 传输的。你享受了 Godot 的易用性，同时拥有了 Steam 的稳定性和防内网穿透能力。