
```GDScript
extends Node
class_name CapabilityComponent

@export var default_sheets : Array[CapabilitySheet]

var tag_blockers: Dictionary[Enums.CapabilityTags, Array] = {}
var default_capabilities: Array = []

func _ready(): 
	_load_sheets(default_sheets)

func _exit_tree(): 
	for cap in _instances: CapabilitySystem.unregister(cap)

# 公开 API：在运行时给角色加/减表单  
func runtime_add_sheet(sheet: CapabilitySheet):  
	_load_sheets([sheet])  
  
func _load_sheets(sheets: Array[CapabilitySheet]):  
	for sheet in sheets:  
		var caps = sheet.instantiate(owner = get_parent())  
		for cap in caps:  
			cap.owner = get_parent()  
			cap.setup()  
			CapabilitySystem.register(cap)  
			_instances.append(cap)

## 阻塞标签
func block_capabilities(tag: Enums.CapabilityTags, capability: BaseCapability) -> void:
	if !tag_blockers.has(tag):
		tag_blockers[tag] = []
	if tag_blockers[tag].has(capability):
		return	
	tag_blockers[tag].append(capability)

func unblock_capabilities(tag: Enums.CapabilityTags, capability: BaseCapability) -> void:
	if !tag_blockers.has(tag):
		return
	tag_blockers[tag].erase(capability)

## 是否阻塞
func is_tag_blocked(tag: Enums.CapabilityTags) -> bool:
	if tag_blockers.has(tag):
		if tag_blockers[tag].size() == 0:
			return false
		else:
			return true
	else:
		return false
```