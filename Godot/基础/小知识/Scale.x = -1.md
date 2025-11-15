https://github.com/godotengine/godot/issues/78613

注意：2D 中，变换矩阵是无法分解出负数的 X 缩放的。由于 Godot 中使用变换矩阵来表示缩放，X 轴上的负数缩放在分解后会变为 Y 轴的负数缩放和一次 180 度的旋转。

在CharacterBody2D中，使用了Scale.x = -1会导致x不会变成-1，反而是一个颠倒操作，导致物体左右来回抽搐。

我看有一种解决方案，通过判断当前朝向状态，确保只会转向一次

```GDScript

var was_facing_right := false
var is_facing_right := false

func _physics_process(delta: float) -> void:
	turn_direction()
	
func turn_direction() -> void:
	var direction = 1 if boss_stat_component.player.global_position.x < owner.global_position.x else -1
	if direction == 1:
		was_facing_right = false
	else:
		was_facing_right = true
	if 	is_facing_right != was_facing_right:
		is_facing_right = was_facing_right
		owner.scale.x = abs(owner.scale.x) * -1

```