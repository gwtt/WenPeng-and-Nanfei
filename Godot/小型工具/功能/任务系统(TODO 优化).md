参考: https://github.com/Skierhou/TaskSystem

实际参与任务系统后才了解其实底层就是一个计数系统，记录的是游戏中出现的所有计数，包括但不限于：消灭了几只怪物，获得了多少金币/材料，提升了多少级，到达某个城镇/位置等等，有的任务需要从任务开始时才计数，而有的任务的计数是累积的，这样我们能将任务大体上分为两类：累积型，递增型。而消灭了几只怪物，获得了多少金币/材料，提升了多少级，到达某个城镇/位置等等实际任务则是游戏中的一个业务标记。

除了这类累积型，和递增型外还有一种是消耗型，如任务条件是消耗100金币或n个xxx道具，既然是消耗型那么一定消耗的"道具"，一切玩家可获得的东西我将它统称为"道具"，详情可参考：[游戏中的背包系统](https://zhuanlan.zhihu.com/p/660125359)。 消耗的是背包中固有的数量，因此可以检测道具存在数量，强制更新给计数器，那么就可以用累积型的记录方式，每次道具数量变化时强制更新计数，或者是在任务的上层逻辑中单独处理这个问题。

还有是任务种类上的区分：主线，分支，日常，周常等等。当然成就也算一种任务，它的完成条件同样来自这个计数系统，是累积型的任务。同样的游戏中除了任务界面可以看到的任务外，一些控制开关的地方也可以使用任务来控制，比如关卡2-3开启需要达成条件：关卡总得星30个以及角色等级达到10级以及通关关卡2-2等一系列条件，如果不在关卡系统上做又想保持通用性就可以使用一个任务来做为开关，这样这个开关的可扩展性可以涉及游戏全局，而不用根据策划需求变化：同时满足角色阶数到达2阶，角色战力达到1w等特殊需求做修改。

### 1. counter_center.gd
```GDScript
extends Node
class_name CounterCenter
# 建议设为 AutoLoad 单例

enum CounterType {
    KILL_MONSTER,
    ITEM_GET,
    ITEM_USE,
    ITEM_NUM,
    ROLE_ATTR_ADD,
    TALK_NPC,
}

# (type, param, new_value, delta, is_increment)
signal counter_changed(counter_type:int, param:int, new_value:int, delta:int, incremental:bool)

# 数据结构 : {CounterType : {param : value}}
var _data:Dictionary = {}

#-------------------- 对外 API --------------------
func add(counter_type:int, delta:int = 1, param:int = 0) -> void:
    _internal_update(counter_type, delta, param, true)

func set(counter_type:int, value:int, param:int = 0) -> void:
    _internal_update(counter_type, value, param, false)

func get(counter_type:int, param:int = 0) -> int:
    return _data.get(counter_type, {}).get(param, 0)

#-------------------- 内部实现 --------------------
func _internal_update(ct:int, value:int, param:int, incremental:bool) -> void:
    if not _data.has(ct):
        _data[ct] = {}
    var old := _data[ct].get(param, 0)
    var new_val:int

    if incremental:
        new_val = old + value
    else:
        new_val = value
        if new_val == old:    # 与当前相同则无需广播
            return

    _data[ct][param] = new_val
    emit_signal("counter_changed", ct, param, new_val, new_val - old, incremental)

```

### 2. task.config.gd

```GDScript
extends Resource
class_name TaskConfig

@export var id:int
@export var name:String
@export_multiline var desc:String

@export var default_open:bool = false
@export var task_type:String = "MAIN"           # MAIN / BRANCH / DAILY / WEEKLY / ACHIEVEMENT / ...
@export var next_ids:Array[int] = []            # 任务链
@export var reward_id:int = 0                   # 奖励表 index

enum TaskRecordType { ACCUMULATE, INCREMENTAL, CONSUMABLE }

# 以下四个数组长度必须一致（可以写自定义 struct 但 Godot 4.2 仍不支持数组内嵌自定义类型序列化）
@export var counter_types:PackedInt32Array      # CounterCenter.CounterType
@export var record_types:PackedInt32Array       # TaskRecordType
@export var max_counts:PackedInt32Array
@export var params:PackedInt32Array             # 怪物 ID / 物品 ID / 等

```

### 3. task_instance.gd
```GDScript
extends Object
class_name TaskInstance

# -------------- signal / enum -----------------
enum Progress { START, IN_PROGRESS, CONDITION_OK, GOT_REWARD }
signal progress_changed(task_id:int, progress_state:int)

# -------------- 变量 --------------------------
var cfg:TaskConfig
var cur:PackedInt32Array      # 当前进度
var got:bool = false          # 已领奖

# -------------- 初始化 ------------------------
func _init(config:TaskConfig):
    cfg = config
    cur = PackedInt32Array()
    cur.resize(cfg.max_counts.size())   # 默认全 0
    CounterCenter.counter_changed.connect(_on_counter_change)

# -------------- 计数回调 ----------------------
func _on_counter_change(ct:int, param:int, new_val:int, delta:int, inc:bool) -> void:
    for i in range(cfg.counter_types.size()):
        if ct != cfg.counter_types[i]:             # 类型不匹配
            continue
        if param != cfg.params[i]:                 # 参数不匹配
            continue

        match cfg.record_types[i]:
            TaskConfig.TaskRecordType.ACCUMULATE:
                cur[i] = new_val                   # 直接取总累计
            TaskConfig.TaskRecordType.INCREMENTAL:
                cur[i] += delta                    # 只关心增量
            TaskConfig.TaskRecordType.CONSUMABLE:
                cur[i] = new_val                   # 先判断够不够，真正扣在领奖时

    _check_finish()

# -------------- 完成判定 ----------------------
func _check_finish() -> void:
    if got:
        return
    var all_ok := true
    for i in range(cur.size()):
        if cur[i] < cfg.max_counts[i]:
            all_ok = false
            break

    var state:int = Progress.CONDITION_OK if all_ok else Progress.IN_PROGRESS
    emit_signal("progress_changed", cfg.id, state)

# -------------- 领奖 --------------------------
func try_get_reward() -> bool:
    if got:
        return false
    if not _is_condition_ok():
        return false

    # 处理消耗型扣除
    for i in range(cfg.record_types.size()):
        if cfg.record_types[i] == TaskConfig.TaskRecordType.CONSUMABLE:
            var item_id:int = cfg.params[i]
            var need:int = cfg.max_counts[i]
            if not Inventory.remove_item(item_id, need):   # 自己实现
                return false

    got = true
    _give_reward()
    emit_signal("progress_changed", cfg.id, Progress.GOT_REWARD)
    return true

func _give_reward() -> void:
    var reward_cfg:RewardConfig = RewardDB.get(cfg.reward_id)   # 根据你的实现
    reward_cfg.apply(PlayerData)

# -------------- 工具 --------------------------
func _is_condition_ok() -> bool:
    for i in range(cur.size()):
        if cur[i] < cfg.max_counts[i]:
            return false
    return true

# -------------- 存档 --------------------------
func to_dict() -> Dictionary:
    return { "id": cfg.id, "cur": cur, "got": got }

static func from_dict(d:Dictionary) -> TaskInstance:
    var config:TaskConfig = TaskDB.get(d["id"])
    var ti := TaskInstance.new(config)
    ti.cur = d["cur"]
    ti.got = d["got"]
    return ti

```

### 4. task_center.gd
```GDScript
# addons/task_system/task_center.gd
extends Node
class_name TaskCenter
# 建议设为 AutoLoad

signal task_added(task_id:int)
signal task_removed(task_id:int)
signal task_progress(task_id:int, progress:int)

var _active:Dictionary     = {}      # id -> TaskInstance
var _ready_to_claim:Array[int] = []  # 条件OK但未领奖
var _rewarded:Array[int]        = [] # 已领奖/闭环

#-------------------- 生命周期 -------------------
func _ready() -> void:
    _open_default_tasks()

func _open_default_tasks() -> void:
    for cfg in TaskDB.get_all():
        if cfg.default_open:
            _open_task(cfg.id)

#-------------------- 任务管理 -------------------
func _open_task(task_id:int) -> void:
    if _active.has(task_id) or _rewarded.has(task_id):
        return
    var cfg:TaskConfig = TaskDB.get(task_id)
    if cfg == null:
        push_error("TaskConfig %s not found!" % task_id)
        return
    var inst := TaskInstance.new(cfg)
    inst.progress_changed.connect(_on_task_progress)
    _active[task_id] = inst
    emit_signal("task_added", task_id)

func _on_task_progress(task_id:int, progress:int) -> void:
    emit_signal("task_progress", task_id, progress)

    match progress:
        TaskInstance.Progress.CONDITION_OK:
            if !_ready_to_claim.has(task_id):
                _ready_to_claim.append(task_id)
        TaskInstance.Progress.GOT_REWARD:
            _on_rewarded(task_id)

func _on_rewarded(task_id:int) -> void:
    _ready_to_claim.erase(task_id)
    _rewarded.append(task_id)
    _active.erase(task_id)
    _open_follow_tasks(task_id)
    emit_signal("task_removed", task_id)

func _open_follow_tasks(task_id:int) -> void:
    var cfg := TaskDB.get(task_id)
    if cfg == null:
        return
    for nid in cfg.next_ids:
        _open_task(nid)

#-------------------- UI / 外部接口 --------------
func get_instance(id:int) -> TaskInstance:
    return _active.get(id, null)

func claim_reward(id:int) -> bool:
    var inst:TaskInstance = _active.get(id, null)
    if inst and inst.try_get_reward():
        return true
    return false

```

### 5. 使用示例  
────────────────────────────────────────

#### 1.案例1

玩家向NPCA接任务，然后去某个地方跟NPCB聊天，然后完成任务，回去领奖励，触发新对话，不能重复接任务的流程。

1. 玩家跟NPC1对话

```GDScript
func _on_player_talk(player:Node):
    CounterCenter.add(CounterCenter.CounterType.TALK_NPC, 1, my_npc_id)
```

需要配置下任务资源实例

```
TaskConfig.tres
---------------
id = 1
name = "替我向学者问好"
desc = "到城北的学者（ID 200）那里报个到，再回来找我。"
default_open = false
task_type = "BRANCH"
next_ids = [2]
reward_id = 7

counter_types  = PoolIntArray( [CounterCenter.CounterType.TALK_NPC] )
record_types   = PoolIntArray( [TaskConfig.TaskRecordType.ACCUMULATE] )
max_counts     = PoolIntArray( [1] )
params         = PoolIntArray( [200] )     # 目标 NPC 的 id

```

2. 脚本示例

A脚本

```GDScript
extends Node3D     # 或需要的节点类型
@export var task_id:int = 1          # 发放的任务 id
@export var npc_id:int = 100         # 自己的 id，用于对话统计（可选）

func _ready():
    pass

func _on_interact():
    var inst := TaskCenter.get_instance(task_id)
    var rewarded := TaskCenter.is_rewarded(task_id)   # 下面会加这个接口
    if inst == null and !rewarded:
        _show_offer_dialog()          # “帮我去找学者聊聊吧”
    elif inst and inst.got == false:
        if inst._is_condition_ok():
            _give_reward()
        else:
            _show_middle_dialog()     # “去找学者了吗？”
    else:
        _show_after_dialog()          # “谢谢你！(已完成，不能再接)”

func _show_offer_dialog():
    DialogUI.show([
        "你好冒险者，能帮我去找学者（北侧 200 号）聊聊吗？",
        { "text":"好的", "action": _accept_task },
        { "text":"我再想想", "action": null }
    ])

func _accept_task():
    TaskCenter.open_task(task_id)     # 见第 D 节
    CounterCenter.add(CounterCenter.CounterType.TALK_NPC, 1, npc_id) # 统计本次对话(可选)

func _give_reward():
    if TaskCenter.claim_reward(task_id):
        DialogUI.show(["干得漂亮！给你一点报酬。"])
    else:
        DialogUI.show(["（似乎出了什么问题，无法领取）"])

```

B脚本

```GDScript
extends Node3D
@export var npc_id:int = 200

func _on_interact():
    CounterCenter.add(CounterCenter.CounterType.TALK_NPC, 1, npc_id)
    DialogUI.show(["哦！原来是那位老朋友让你来的，替我向他问好！"])

```

3. 防止重复领取

```GDScript
# task_center.gd（追加）

func is_active(id:int) -> bool:
    return _active.has(id)

func is_rewarded(id:int) -> bool:
    return _rewarded.has(id)

# 允许外部主动开启，但内部仍会检查重复
func open_task(id:int) -> void:
    _open_task(id)

```

NPC_A 在对话时通过 `TaskCenter.is_active / is_rewarded` 先行判断，就不会重复发送任务；TaskCenter 内部的 `_open_task()` 本身也有双重保护，不会让同一任务重复进入 active。

1. 玩家与 npc_a 交互 → 触发 `_show_offer_dialog()`  
    • 玩家点击“好的” → TaskCenter.open_task(1) → TaskInstance 被创建  
    • npc_a 对话 UI 关闭
    
2. 玩家去找 npc_b → 与其交互  
    • npc_b 脚本调用 `CounterCenter.add(TALK_NPC, 1, 200)`  
    • TaskInstance #1 收到计数，达成条件 → progress_changed(CONDITION_OK) → TaskCenter 把任务列入待领奖
    
3. 玩家回到 npc_a 再次交互  
    • 检测到任务已 CONDITION_OK，执行 `_give_reward()`  
    • TaskCenter.claim_reward(1) → TaskInstance.try_get_reward() → 发奖、Progress=GOT_REWARD  
    • TaskCenter 监听到 GOT_REWARD → 移出活跃列表、加入已领奖列表、自动开启 next_ids（如有）
    
4. 后续 npc_a 的对话因 `TaskCenter.is_rewarded(1)` 为 true，只显示“已完成”台词，无法再次领取该任务。
#### 2.案例2

1. 怪物死亡时

```GDSCRIPT
CounterCenter.add(CounterCenter.CounterType.KILL_MONSTER, 1, monster_id)
```

2. 玩家获得或失去道具时

```GDScript
CounterCenter.add(CounterCenter.CounterType.ITEM_GET, amount, item_id) CounterCenter.set(CounterCenter.CounterType.ITEM_NUM, new_total, item_id)
```

3. 玩家点击领取任务奖励

```GDScript
TaskCenter.claim_reward(task_id)
```

4. UI 订阅

```GDScript
TaskCenter.task_added.connect(_on_task_added)
TaskCenter.task_progress.connect(_on_task_progress)
```

