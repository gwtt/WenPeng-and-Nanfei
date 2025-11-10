[[架构大纲图]]
[[案例 1：玩家死亡流程]]
[[案例 2：交互拾取宝箱]]
把一连串需要按照先后顺序发生的动作（动画、特效、等待、回调……）排成队列，逐帧执行、自动等待，无需你去写 yield / await 或嵌套回调。

```GDScript
extends Node
class_name ActionQueue

func empty() -> void:
	pass
	
func idle(duration: float) -> void:
	pass
	
func event(function_owner: Node, callback: Callable) -> void:
	pass
	
func duration(duration: float, function_owner: Node, callback: Callable) -> void:
	pass

func capability(capability_owner: Node, sub_class: ActionCapability) -> void:
	pass

func set_looping(b_looping: bool) -> void:
	pass


```