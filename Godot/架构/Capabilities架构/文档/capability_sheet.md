```GDScript
extends Node
class_name CapabilitySheet

@export var component_scenes : Array[PackedScene]           
@export var capability_scripts : Array[Script]              
@export var nested_sheets : Array[CapabilitySheet]

func instantiate(owner: Node) -> Array[BaseCapability]:
	var caps: Array[BaseCapability] = []
	
	# 1. 场景加载
	for scene in component_scenes: 
		var node_name := scene.resource_path.get_file().get_basename() 
		if owner.has_node(node_name): # 简单去重 
			continue 
		var node := scene.instantiate() 
		node.name = node_name 
		owner.add_child(node)

	# 2. 创建 Capability 实例
	for scr in capability_scripts:
		var cap: BaseCapability = scr.new()
		caps.append(cap)

	# 3. 递归嵌套 Sheet
	for sheet in nested_sheets:
		caps.append_array(sheet.instantiate(owner))

	return caps


```
用途：
- 描述结构
- 统一实例化
- 复用+组合，比如BossSheet = 近战怪 Sheet + 狂暴 Sheet + 飞行 Sheet。
- 编辑友好
