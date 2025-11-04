
```GDScript
extends Node
class_name CapabilityComponent

@export var default_sheets : Array[CapabilitySheet]

var tag_blockers: Dictionary[Enums.CapabilityTags, Array] = {}
var default_capabilities: Array = []

func _ready(): 
	_load_sheets(default_sheets)
	
func runtime_add_sheet(sheet: CapabilitySheet):  
	_load_sheets([sheet])  
  
func _load_sheets(sheets: Array[CapabilitySheet]):  
	for sheet: CapabilitySheet in sheets:  
		var caps = sheet.instantiate(get_parent())  
		for cap: BaseCapability in caps:  
			cap.owner = get_parent()  
			cap.setup()  
			CapabilitySystem.register(cap)  
			default_capabilities.append(cap)
			
## 阻塞标签
func block_capabilities(tag: Enums.CapabilityTags, capability: BaseCapability) -> void:
	if !tag_blockers.has(tag):
		tag_blockers[tag] = []
		return
	if tag_blockers[tag].has(capability):
		return	
	tag_blockers[tag].append(capability)

func unblock_capabilities(tag: Enums.CapabilityTags, capability: BaseCapability) -> void:
	if !tag_blockers.has(tag):
		return
	tag_blockers[tag].erase(capability)

## 是否阻塞
func is_block(tag: Enums.CapabilityTags) -> bool:
	if tag_blockers.has(tag):
		if tag_blockers[tag].size() == 0:
			return false
		else:
			return true
	else:
		return false

```