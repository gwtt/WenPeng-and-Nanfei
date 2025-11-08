
```GDScript
extends Node
class_name BaseCapability

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
	CapabilitySystem.register(self)

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
	CapabilitySystem.unregister(self)

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

1. 当GameObject Spawned时，我们在该GameObject上的Capabilities上运行`Setup`，通常用于初始化本地成员变量，比如从GameObject上获取Component的引用/指针
2. 当某个Capability处于非激活状态（Inactive）时，我们**每帧检查**`ShouldActivate`
3. 在某个时刻，`ShouldActivate`返回`true`，这个Capability变为激活状态（Active），我们运行`OnActivated`
4. 当这个Capability处于激活状态时，我们改为**每帧检查**`ShouldDeactivate`，并且**每帧运行**`TickActive`
5. 当`ShouldDeactivate`返回`true`时，我们运行`OnDeactivated`，然后回到**每帧检查**`ShouldActivate`