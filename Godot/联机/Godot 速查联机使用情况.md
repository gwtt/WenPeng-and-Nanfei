这是 Godot 多人游戏开发的**核心决策清单**。

记住一个心法：
1. **状态同步靠节点 (MultiplayerSynchronizer)
2. 一次性事件靠 RPC
3. 逻辑控制靠 Authority 判断。

---

### 1. 什么时候用 RPC？(处理“事件”)

RPC (Remote Procedure Call) 适用于**“瞬间发生、触发一次”**的事情。

#### A. @rpc("authority", "call_local", "reliable")

**场景：服务器下达的关键指令（必须送达，不能丢包）**

- **谁调用**：服务器 (Server)。
    
- **谁执行**：所有人 (Server + Clients)。
    
- **典型案例**：
    
    - 游戏开始/结束。
        
    - 玩家重生 / 死亡。
        
    - 打开一扇门 / 触发机关。
        
    - 背包里增加了一个关键道具。
        
    - 播放一个**涉及逻辑变化**的动画（如 Boss 进入二阶段变身）。
        

#### B. @rpc("authority", "call_local", "unreliable")

**场景：视觉特效、不影响游戏进程的反馈（丢包也无所谓）**

- **谁调用**：服务器 (Server)。
    
- **谁执行**：所有人。
    
- **典型案例**：
    
    - **飘血数字**（就是你刚才做的 play_hurt_visuals）。
        
    - **受击闪烁**。
        
    - **播放音效**。
        
    - **非关键粒子的生成**（脚下的灰尘、撞墙的火花）。
        
- 注意：用 unreliable 可以节省带宽，防止网络拥堵导致游戏卡顿。
    

#### C. @rpc("any_peer", "call_local", "reliable")

**场景：客户端向服务器“请愿” / 发送输入**

- **谁调用**：客户端 (Client)。
    
- **谁执行**：主要是服务器（但加了 call_local 客户端自己也会先跑一遍，用于“客户端预测”）。
    
- **典型案例**：
    
    - 玩家按下“攻击”键（请求攻击）。
        
    - 玩家点击“购买”按钮。
        
    - 发送聊天信息。
        
- 注意：这类函数内部必须做安全检查（防作弊），因为黑客可以随意调用它。
    

---

### 2. 什么时候用 if not is_multiplayer_authority(): return？

这句话的意思是：**“如果我不是管事人，我就闭嘴/不干活”**。它用于**持续运行的逻辑**中。

#### A. 在 _physics_process 或 _process 中

**场景：防止逻辑重复计算**

- **移动逻辑**：
    
    codeGdscript
    
    ```
    func _physics_process(delta):
        # 只有这个角色的控制者（服务器或玩家自己）才计算移动
        if not is_multiplayer_authority(): return 
        
        velocity = Input.get_vector(...) * speed
        move_and_slide()
    ```
    
- **AI 逻辑**：
    
    codeGdscript
    
    ```
    func _physics_process(delta):
        # 只有服务器负责 AI 寻路，客户端只负责同步位置
        if not is_multiplayer_authority(): return
        
        nav_agent.get_next_path_position() ...
    ```
    

#### B. 在 信号回调 (_on_area_entered) 中

**场景：防止重复判定**

- **造成伤害**：
    
    codeGdscript
    
    ```
    func _on_area_entered(area):
        # 只有服务器才有资格判伤害、扣血
        if not is_multiplayer_authority(): return
        
        # 扣血逻辑...
    ```
    
- 如果不加这句，客户端也会运行扣血逻辑，导致飘血数字飘两次，或者客户端血条先扣了，服务器还没扣，导致不同步。
    

---

### 3. 什么时候用 MultiplayerSynchronizer 节点？

适用于**“持续变化的数值”**，不要用 RPC 去每帧发位置！

#### A. 基础属性同步

**场景：每一帧都在变的数据**

- **位置 (position)**、**旋转 (rotation)**。
    
- **动画混合树参数** (animation_tree["parameters/blend_position"])。
    
- **简单状态开关** (is_jumping, is_sprinting)。
    

#### B. 游戏数值同步

**场景：需要所有端保持一致的关键数值**

- **当前生命值 (current_hp)**。
    
- **无敌状态 (is_invincible)**。
    

---

### ⚡ 终极速查表

|                     |                |                                               |
| ------------------- | -------------- | --------------------------------------------- |
| 需求                  | 推荐机制           | 代码/配置                                         |
| **角色移动 (Position)** | **同步器**        | MultiplayerSynchronizer (Watch: position)     |
| **当前血量 (HP)**       | **同步器**        | MultiplayerSynchronizer (Watch: hp)           |
| **玩家点击攻击键**         | **RPC (上行)**   | @rpc("any_peer", "call_local", "reliable")    |
| **判定伤害/扣血**         | **普通函数 + 权限锁** | func take_damage(): if not authority: return  |
| **飘字/特效/音效**        | **RPC (下行)**   | @rpc("authority", "call_local", "unreliable") |
| **玩家死亡/重生**         | **RPC (下行)**   | @rpc("authority", "call_local", "reliable")   |
| **AI 寻路/Boss行为**    | **普通函数 + 权限锁** | _process(): if not authority: return          |

### 常见误区警示 ⚠️

1. **在 _process 里写 RPC**：❌ 绝对禁止。这会每秒发60次网络包，瞬间炸服。请用 MultiplayerSynchronizer。
    
2. **在 RPC 里写 if not authority: return**：❌ 错了。
    
    - **普通函数**：写这个是为了防止客户端乱跑逻辑。
        
    - **RPC 函数**：通常是用来通知对方执行的，里面应该是纯视觉或纯状态赋值逻辑。
        
3. **Visuals 依赖 Logic**：❌ 比如“因为扣血了所以自动飘字”。
    
    - ✅ 正解：逻辑扣血完，显式调用 play_visual.rpc()。这样解耦最安全。


### 假设你做了一个双人游戏：

- **服务器 (Server)**：ID = 1
    
- **玩家 A (Client A)**：ID = 100 (假设)
    
- **玩家 B (Client B)**：ID = 200 (假设)
    

#### 场景一：世界里的怪物 (Enemy)

怪物的逻辑通常由服务器运算。

- **设置**：所有端上的怪物节点，set_multiplayer_authority(1)。
    

|   |   |   |   |   |
|---|---|---|---|---|
|运行环境|节点的 Authority 属性|当前环境的 Unique ID|is_multiplayer_authority()|结果|
|**服务器**|1|1|1 == 1|**True (跑逻辑)**|
|**玩家 A**|1|100|1 == 100|**False (只看不动)**|
|**玩家 B**|1|200|1 == 200|**False (只看不动)**|

#### 场景二：玩家 A 的角色 (Player_A)

这是玩家 A 控制的角色。

- **设置**：当角色生成时，服务器执行 player_a_node.set_multiplayer_authority(100)，这个设置会自动同步给所有客户端（如果使用了 MultiplayerSpawner）。
    

|          |                  |                 |                            |                                 |
| -------- | ---------------- | --------------- | -------------------------- | ------------------------------- |
| 运行环境     | 节点的 Authority 属性 | 当前环境的 Unique ID | is_multiplayer_authority() | 结果                              |
| **服务器**  | 100              | 1               | 100 == 1                   | **False** (通常服务器不直接控制玩家移动，而是校验) |
| **玩家 A** | 100              | 100             | 100 == 100                 | **True (读取键盘输入)**               |
| **玩家 B** | 100              | 200             | 100 == 200                 | **False (只看同步过来的位置)**           |

### 假如做一个肉鸽联机游戏，玩家1是主机，玩家2发射了一颗子弹，对boss造成伤害

#### 1. 开火阶段 (Input & Spawn)

- **P2 (客机)**：按下攻击键 -> 本地播放枪口火光/音效（为了手感） -> **发 RPC 请求**给 P1：“我要在坐标(x,y)朝方向(v)发射子弹”。
    
- **P1 (主机)**：收到请求 -> 校验（没冷却？有子弹？） -> **生成“真子弹”节点**。
    
- **同步**：P1 的 MultiplayerSpawner 自动把这颗子弹同步给 P2（和 P3, P4），P2 屏幕上出现飞行的子弹。
    

#### 2. 命中阶段 (Collision)

- **P1 (主机)**：**真子弹**在 P1 的电脑上飞 -> 撞到 Boss 的碰撞体。
    
- **P1 (主机)**：触发 area_entered -> 确认打中的是 Boss。
    
- **P2 (客机)**：屏幕上的子弹只是在飞（仅仅是影像），**不处理**任何碰撞逻辑。
    

#### 3. 结算阶段 (Damage)

- **P1 (主机)**：计算伤害 (攻击 - 防御) -> 修改 Boss 血量变量 current_hp -= damage。
    
- **P1 (主机)**：判断 Boss 是否死亡。
    

#### 4. 反馈阶段 (Feedback)

- **P1 (主机)**：
    
    1. 通过 MultiplayerSynchronizer 自动把 Boss 的**新血量**同步给 P2（P2 血条更新）。
        
    2. 发送 **RPC 广播**：“Boss 受伤了，伤害值是 100”。
        
- **P2 (客机)**：收到 RPC -> 在 Boss 头顶弹出“100”的伤害飘字 -> 播放受击音效。

`所以所有的计算判断都在主机上操作是吧，然后boss的所有属性和操作也是在主机上操作。`


主机决定了Boss的下一步操作和技能操作。然后用MultiplayerSynchronizer来同步Boss的视觉效果。

主机决定了所有对象的属性和操作。比如Boss的血量、p1和p2的血量。

主机决定子弹的碰撞和检测，p2那边显示就是一个模型和图片。
