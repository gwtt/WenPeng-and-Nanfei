
```GDScript
extends Capability
class_name PlayerMoveCap

var _stats : StatsComponent

func setup():
	_stats = owner.get_node("StatsComponent")

func tick_active(delta):
	if is_blocked(Enums.CapabilityTags.MOVE):
		return
	var dir = Vector2(
		Input.get_action_strength("ui_right") - Input.get_action_strength("ui_left"),
		Input.get_action_strength("ui_down")  - Input.get_action_strength("ui_up"))
	if dir != Vector2.ZERO:
		owner.position += dir.normalized()*_stats.move_speed*delta

```