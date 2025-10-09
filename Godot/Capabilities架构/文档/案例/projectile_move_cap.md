```GDScript
extends Capability
class_name ProjectileMoveCap

var velocity : Vector2 = Vector2.ZERO

func tick_active(delta):
	owner.position += velocity*delta
	if !get_viewport().get_visible_rect().grow(32).has_point(owner.position):
		owner.queue_free()

```