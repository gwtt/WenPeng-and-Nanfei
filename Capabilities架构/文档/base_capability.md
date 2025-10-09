
```GDScript
extends Node
class_name BaseCapability

@export var capability_component: CapabilityComponent

var tags = []
var tick_group: Enums.ETickGroup = Enums.ETickGroup.GamePlay
var tick_group_order = 100
var active := false;

# 激活持续时间
var active_duration: float = 0
# 非激活持续时间
var deactive_duration: float = 0

func _ready() -> void:
	set_up()
	owner.tree_exiting.connect(on_owner_destroyed)

# GameObject实例化时启动, 主动激活
func set_up() -> void:
	CapabilityManager.register(self)

# 激活状态时每帧检查
func should_activate() -> bool:
	return true

# 非激活状态时每帧检查
func should_deactivate() -> bool:
	return false

# 激活时运行
func on_active() -> void:
	pass

# 非激活时运行
func on_deactivate() -> void:
	pass

# 激活时,每帧运行
func tick_active(_delta_time: float) -> void:
	pass

# 拥有者摧毁时
func on_owner_destroyed():
	if (active):
		on_deactivate()
	CapabilityManager.unregister(self)

# 切换激活状态,返回切换后的值
func toggle() -> bool:
	if should_activate() && active == false:
		active = true
		on_active()
		return true

	if should_deactivate() && active == true:
		active = false
		on_deactivate()
		return false

	if should_deactivate() && active == false:
		return false

	if should_activate() && active == true:
		return true

	return false
```