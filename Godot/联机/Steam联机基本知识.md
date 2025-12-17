## 1. 核心身份：Steam ID vs Peer ID

这是联机的基石。

### What (是什么)

- Steam ID (64位整数): 你的身份证号。全球唯一，永久不变。

- Peer ID (整数): 你的房间号。每次进游戏房间生成的临时 ID。

- 1: 永远代表 主机 (Host/Server)。

- >1 (随机数): 代表 客户端 (Client)。

### Why (为什么分两个)

- Godot 底层只需要简单的数字 (Peer ID) 来通讯，处理速度快。

- Steam ID 用于邀请好友、验证正版，进房间后就不常用了。

### How (怎么用)

你需要维护一个字典来把它们对应起来：

# 例如在 Lobby.gd 或 GameState.gd 中

var peer_map = {} # { peer_id : steam_id }

func _on_player_join(steam_id):

    # Steam接口会自动处理连接，Godot会分配一个 peer_id

    pass

# 重点：发消息时只认 Peer ID

rpc_id(target_peer_id, "my_function") 

---

## 2. RPC (Remote Procedure Call)

### What (是什么)

远程遥控。你在你的电脑上调用一个函数，让它在别人的电脑上执行。

### Why (为什么用)

处理一次性事件。比如：开枪、播放音效、受到伤害、发送聊天信息。

### How (怎么用)

在你的 GunAttackComponent.gd 中：

# 1. 声明：any_peer允许任何人调用，call_local允许自己也运行一遍

@rpc("any_peer", "call_local")

func fire_bullet_rpc(pos: Vector2, dir: Vector2):

    # 这里的代码会在所有人的电脑上跑

    var bullet = bullet_scene.instantiate()

    bullet.position = pos

    # ... 发射逻辑 ...

    print("有人开枪了！")

# 2. 调用：当你按下鼠标时

func _on_attack():

    # 告诉所有人（包括我自己）：我开枪了

    fire_bullet_rpc.rpc(global_position, direction)

---

## 3. rpc_id (定向 RPC)

### What (是什么)

悄悄话。只发给特定的某一个人（或只发给主机）。

### Why (为什么用)

- 私聊。

- 服务器授权验证（客户端告诉主机“我要买东西”，主机扣钱后告诉该客户端“购买成功”）。

- 节省带宽（不需要所有人都知道的事）。

### How (怎么用)

# 客户端想向主机汇报伤害

func apply_damage(dmg):

    if multiplayer.is_server():

        # 主机直接扣血

        hp -= dmg

    else:

        # 客户端告诉主机(ID为1)："我打中怪了，请扣血"

        request_damage.rpc_id(1, dmg)

@rpc("any_peer") 

func request_damage(dmg):

    # 这段代码只在主机运行

    if multiplayer.is_server():

        hp -= dmg

---

## 4. MultiplayerSpawner (自动生娃机)

### What (是什么)

自动同步节点的创建和销毁。

### Why (为什么用)

如果你用 RPC 手动 instantiate 子弹或怪物，你需要处理复杂的 ID 同步问题。Spawner 会自动帮你在所有客户端生成完全一样的节点，名字都一样。

### How (怎么用)

1. 添加节点: 在场景里加一个 MultiplayerSpawner 节点。

2. 设置 Spawn Path: 指向你会把子弹/怪物 add_child 进去的父节点（比如 Level/Projectiles）。

3. 配置列表: 在 Inspector 里把你的 Bullet.tscn 加入 Auto Spawn List。

4. 代码 (只在主机写):

# 只需主机执行，Spawner 会自动在所有客户端生成这个子弹

if multiplayer.is_server():

    var bullet = bullet_scene.instantiate()

    parent_node.add_child(bullet) 

    # 完事了！Godot 自动处理剩下的。

---

## 5. MultiplayerSynchronizer (自动同步机)

### What (是什么)

持续同步变量。

### Why (为什么用)

有些东西是时刻在变的，比如：位置、旋转、血量。用 RPC 一直发太麻烦且由于网络波动会卡顿。

### How (怎么用)

1. 添加节点: 给 Player 或 Enemy 场景加一个 MultiplayerSynchronizer。

2. 设置权限:

- 如果是玩家：Authority 是该玩家的 Peer ID。

- 如果是怪物：Authority 是主机 (1)。

1. 配置属性: 点击底部的 "Replication" 面板 -> Add Property -> 选择 position, rotation, health 等。

- Spawn: 生成时同步一次。

- Watch: 只要变了就同步 (适合血量)。

- Always: 一直同步 (适合位置，但也最费流量)。

---

## 总结：该用哪个？

|场景|推荐工具|例子|
|---|---|---|
|我要告诉所有人发生了一件事|RPC|开枪、跳跃、死亡动画|
|我要只告诉主机一件事|rpc_id(1)|请求购买物品、验证击中|
|我要在场景里生成一个物体|MultiplayerSpawner|生成子弹、掉落物、刷怪|
|物体在移动，需要大家看到一致|MultiplayerSynchronizer|玩家移动同步|
|数值变化 (血量/弹药)|MultiplayerSynchronizer|血条同步|

### 极简代码模板 (Player.gd)

extends CharacterBody2D

# 设置谁有权控制这个角色

func _enter_tree():

    # 比如这个节点名是 "12345"，就把权限给 ID 为 12345 的玩家

    set_multiplayer_authority(str(name).to_int())

func _physics_process(delta):

    # 只有我是这个角色的主人，我才能控制移动

    if is_multiplayer_authority():

        velocity = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down") * 400

        move_and_slide()

    # 只要有了 MultiplayerSynchronizer 同步 position，

    # 别人电脑上的这个角色就会自动跟着动。