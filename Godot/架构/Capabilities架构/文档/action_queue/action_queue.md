[[架构大纲图]]
[[案例 1：玩家死亡流程]]
[[案例 2：交互拾取宝箱]]
把一连串需要按照先后顺序发生的动作（动画、特效、等待、回调……）排成队列，逐帧执行、自动等待，无需你去写 yield / await 或嵌套回调。

```GDScript
extends Node
class_name ActionQueue
signal finished                                # 所有 Action 执行完毕

var _queue : Array = []
var _running := false
var _owner : Node = null                      # 可选：队列所属者

func _init(owner := null):
    _owner = owner

#--------------------------------------------------
# 公开 API
#--------------------------------------------------
func do(fn : FuncRef) -> void:
    _queue.push_back({type = "call", fn = fn})
    _kick()

func wait(seconds : float) -> void:
    _queue.push_back({type = "wait", time = seconds})
    _kick()

func until(target : Object, signal_name : String = "") -> void:
    _queue.push_back({type = "await", target = target, sig = signal_name})
    _kick()

func clear() -> void:
    _queue.clear()
    _running = false

#--------------------------------------------------
# 内部流程
#--------------------------------------------------
func _kick() -> void:
    if not _running and _queue.size() > 0:
        _running = true
        _step()

func _step() -> void:
    if _queue.empty():
        _running = false
        emit_signal("finished")
        return

    var action = _queue.pop_front()
    match action.type:
        "call":
            action.fn.call_func()
            _step()

        "wait":
            var t := Timer.new()
            t.wait_time = action.time
            t.one_shot = true
            t.connect("timeout", self, "_on_timer_done", [t])
            add_child(t)
            t.start()

        "await":
            var tgt := action.target
            var sig := action.sig if action.sig != "" else "finished"
            tgt.connect(sig, self, "_on_signal_done", [], CONNECT_ONE_SHOT)

func _on_timer_done(timer : Timer) -> void:
    timer.queue_free()
    _step()

func _on_signal_done() -> void:
    _step()

```