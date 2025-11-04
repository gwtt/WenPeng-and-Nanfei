
```GDScript
extends Node

## 所有的组件管理
var all_capabilities = {}

func _ready() -> void:
	for group in Enums.ETickGroup.values():
		all_capabilities[group] = []

func register(capability: BaseCapability):
	var array: Array = all_capabilities[capability.tick_group]
	array.append(capability)

func unregister(capability: BaseCapability):
	var array: Array = all_capabilities[capability.tick_group]
	array.erase(capability)

## 每帧去运行,切换后如果是激活状态就运行
func _physics_process(delta: float) -> void:
	for tick_group in all_capabilities.keys():
		for capability: BaseCapability in all_capabilities.get(tick_group):
			if capability.toggle():
				capability.tick_active(delta)
```