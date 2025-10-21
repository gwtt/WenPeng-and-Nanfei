[游戏中的背包](https://zhuanlan.zhihu.com/p/660125359)

记录下如何实现简易的格子型背包+合成系统。

流程上：
物品获取 → 尝试装备栏 → 尝试背包 → 合成检查 → UI 更新 → 数据保存
### 1. 物品设计

因为简单化，我们就假设物品的属性有以下几个：
- id 做映射
- name 显示用
- icon 显示用
- max_stack 最大堆叠
- type 类型
- 装备属性
- 消耗属性

```GDScript
extends Resource
class_name ItemData

# =========== 基本信息 ===========
@export var id          : String              # 全局唯一，用来存档
@export var name        : String              # 在 UI 中显示
@export var icon        : Texture2D           # 背包格子用
@export_range(1, 999) var max_stack : int = 1 # >1 表示可叠加

# =========== 类型 ===========
enum ItemType { MATERIAL, EQUIPMENT, CONSUMABLE }
@export var type : ItemType = ItemType.MATERIAL

# =========== 可选——装备字段 ===========
@export var equip_slot : String = ""  # “weapon”“head”“body”…  留空则不是装备
@export var atk  : int = 0
@export var def  : int = 0

# =========== 可选——消耗品字段 ===========
@export var hp_restore : int = 0
@export var mp_restore : int = 0

# ------------ 辅助 ------------
func is_stackable() -> bool:
    return max_stack > 1

```

### 2. 装备栏设计

```GDScript
extends Node
class_name EquipmentBar

# ----------- 信号 -----------
signal changed    # 穿戴或脱下成功后发，供 UI 刷新

# ----------- 槽位名字 -----------
const SLOT_WEAPON     := "weapon"
const SLOT_HEAD       := "head"
const SLOT_BODY       := "body"
const SLOT_FEET       := "feet"
const SLOT_ACCESSORY  := "accessory"

# 填在这里的字符串决定总体拥有多少槽
const ALL_SLOTS := [SLOT_WEAPON, SLOT_HEAD, SLOT_BODY, SLOT_FEET, SLOT_ACCESSORY]

# ----------- 数据结构 -----------
# { "weapon": ItemStack, ... }    为空用 null
var slots : Dictionary = {}

func _ready():
    # 初始化为空
    for s in ALL_SLOTS:
        slots[s] = null


# =================================================================
# 1. 查询
# =================================================================
func has_item(slot: String) -> bool:
    return slots.get(slot) != null

func get_item(slot: String) -> ItemStack:
    return slots.get(slot, null)

func is_valid_slot_name(slot: String) -> bool:
    return ALL_SLOTS.has(slot)


# =================================================================
# 2. 穿戴
# =================================================================
# 传入一个 ItemStack（count 必须==1），成功返回 true
func try_equip(stack: ItemStack) -> bool:
    if not stack:
        return false
    var data: ItemData = stack.item
    # ① 必须是装备
    if data.type != ItemData.ItemType.EQUIPMENT:
        return false
    # ② 判断槽位名
    var slot := data.equip_slot
    if not is_valid_slot_name(slot):
        return false
    # ③ 该槽必须为空
    if slots[slot]:
        return false
    # ④ 写入
    slots[slot] = stack
    emit_signal("changed")
    return true


# =================================================================
# 3. 脱下
# =================================================================
# 返回脱下的 ItemStack 或 null
func unequip(slot: String) -> ItemStack:
    if not is_valid_slot_name(slot):
        return null
    var stack: ItemStack = slots[slot]
    if stack:
        slots[slot] = null
        emit_signal("changed")
    return stack


# =================================================================
# 4. 统计属性
# =================================================================
func total_attack() -> int:
    var atk := 0
    for s in ALL_SLOTS:
        var st: ItemStack = slots[s]
        if st:
            atk += st.item.atk
    return atk

func total_defence() -> int:
    var d := 0
    for s in ALL_SLOTS:
        var st: ItemStack = slots[s]
        if st:
            d += st.item.def
    return d

```

### 3. 背包设计

```GDScript
extends Node
class_name Inventory

# ------------- 信号 -------------
signal changed                          # 内容有变动时发

# ------------- 参数 -------------
@export_range(1, 200) var capacity := 20    # 总格子数，可在 Inspector 调
@export var allow_overflow := false         # 调试用，真溢出则直接塞末尾

# ------------- 数据结构 -------------
# 数组长度固定；空位存 null，ItemStack.count 绝不会为 0
var slots : Array = []

func _ready() -> void:
    _create_empty()

func _create_empty():
    slots = []
    for i in capacity:
        slots.append(null)


# =================================================================
# 查询 / 工具
# =================================================================
func is_full() -> bool:
    return _first_free_index() == -1

func total_used_slots() -> int:
    var n := 0
    for s in slots:
        if s:
            n += 1
    return n

func _first_free_index() -> int:
    for i in slots.size():
        if not slots[i]:
            return i
    return -1

# 同 id 且未满的格
func _first_stackable_index(data: ItemData) -> int:
    for i in slots.size():
        var st: ItemStack = slots[i]
        if st and st.item.id == data.id and st.count < st.item.max_stack:
            return i
    return -1


# =================================================================
# 1. 添加物品
# =================================================================
# 返回还没放进去的数量（0 则全部放入）
func add_item(data: ItemData, amount: int = 1) -> int:
    var left := amount
    # ① 尝试合并到已有叠
    while left > 0:
        var idx := _first_stackable_index(data)
        if idx == -1:
            break
        var st: ItemStack = slots[idx]
        var room := st.item.max_stack - st.count
        var take := min(room, left)
        st.count += take
        left -= take
    # ② 放到空位
    while left > 0:
        var idx_free := _first_free_index()
        if idx_free == -1:
            break
        var take := min(data.max_stack, left)
        slots[idx_free] = ItemStack.new(data, take)
        left -= take
    # ③ 触发事件
    if left != amount:
        emit_signal("changed")
    # ④ 可选溢出
    if allow_overflow and left > 0:
        push_warning("Inventory overflow (%d) – dropped." % left)
        left = 0
    return left


# =================================================================
# 2. 从指定格子取出 / 拆分
# =================================================================
# 返回一个新的 ItemStack（可能比请求的数量少），原格子里相应减少
func take_from_slot(index: int, amount: int = 1) -> ItemStack:
    if !_is_valid_index(index):
        return null
    var st: ItemStack = slots[index]
    if not st:
        return null
    var take := clamp(amount, 1, st.count)
    var new_stack := ItemStack.new(st.item, take)
    st.count -= take
    if st.count == 0:
        slots[index] = null
    emit_signal("changed")
    return new_stack


# =================================================================
# 3. 直接塞入一个 ItemStack（拖拽用）
# =================================================================
# 成功 true，失败 false
func try_place_stack(index: int, stack: ItemStack) -> bool:
    if not stack or !_is_valid_index(index):
        return false
    var curr: ItemStack = slots[index]
    # 空格，直接放
    if not curr:
        slots[index] = stack
        emit_signal("changed")
        return true
    # 同物品且可叠加
    if curr.item.id == stack.item.id and curr.item.is_stackable():
        var room := curr.item.max_stack - curr.count
        if room == 0:
            return false
        var take := min(room, stack.count)
        curr.count += take
        stack.count -= take
        # 如果搬空了 stack 直接 ok
        if stack.count == 0:
            emit_signal("changed")
            return true
        return false
    # 不同物品 → 交换
    slots[index] = stack
    emit_signal("changed")
    return true


# =================================================================
# 4. 交换两个格子
# =================================================================
func swap_slots(a: int, b: int) -> bool:
    if not _is_valid_index(a) or not _is_valid_index(b) or a == b:
        return false
    var tmp = slots[a]
    slots[a] = slots[b]
    slots[b] = tmp
    emit_signal("changed")
    return true


# =================================================================
# 内部
# =================================================================
func _is_valid_index(i: int) -> bool:
    return i >= 0 and i < slots.size()

```

### 4. 合成表设计

```GDScript
extends Resource
class_name RecipeData
@export var ingredients : Array[String]   # 参与合成的 item.id，长度 2～5
@export var result_id   : String          # 产物 item.id
@export var result_cnt  : int = 1
```

ingredients = ["iron_ore", "coal"] 
result_id = "iron_ingot"