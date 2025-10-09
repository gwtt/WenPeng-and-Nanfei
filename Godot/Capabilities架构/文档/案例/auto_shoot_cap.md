```GDScript
extends Capability
class_name AutoShootCap

@export var projectile_scene : PackedScene
var _timer := 0.0
var interval := 0.8

func setup():
	if projectile_scene == null:
		projectile_scene = load("res://scenes/projectile.tscn")

func tick_active(delta):
	if is_blocked(Enums.CapabilityTags.SHOOT):
		return
	_timer -= delta
	if _timer <= 0.0:
		_timer = interval
		_fire()

func _fire():
	for i in 4:
		var p := projectile_scene.instantiate()
		p.global_position = owner.global_position
		get_tree().current_scene.add_child(p)
		# 给子弹设置速度
		p.call("set_velocity", Vector2.RIGHT.rotated(i*PI/2.0)*400)

```