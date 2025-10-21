生命周期是必不可少的概念。如果不了解生命周期，必定会遇到在执行_ready的时候，遇到某某manager为空的问题，然后这个时候你只会使用
await get_tree().create_timer(1).timeout这种丑陋的方法来解决。

对于一个Node节点（万物之始）

## `_init()`

Godot 在创建时调用此函数，无论您是创建节点（通过编写 `var node ：= Node.new（）`）还是节点来自实例化。

当然可以重载_init()函数，使用new()的时候，必须强制传递。但是这种情况下的话，Godot引擎将无法自动实例化它。

所以一般情况下，保留默认的_init()函数。

## `_enter_tree()`

Godot被添加到场景树的时候调用这个函数，具体时间是使用add_child(node)之后立刻调用此函数。

如果一个节点有子节点，那么所有子节点的 `_enter_tree（）` 都将递归调用，直到最后一个孙子节点。当一个节点进入树时，它的父节点会发出`一个 child_entered_tree` 信号。

## `_ready()`

这是你创建一个脚本就会默认带的方法。这个函数首先在孙子中被调用，然后沿着树_向上_走，直到根。这意味着当调用 `_ready（）` 时，所有子级也都准备好了。

`_ready（）` 的功能与 `_enter_tree（）` 相反：`_enter_tree（）` 从父节点从上往下执行，而 `_ready（）` 从最末端的子节点开始并往上，直到父节点。

当节点完成其 `_ready（）` 函数运行时，它会分派`ready`信号。

## `_process()` and `_physics_process()`  

这两个函数在不同的时刻运行，具体取决于帧速率。默认情况下，_process（）函数将无上限运行，而 _physics_process（）将以 60fps 运行。

这两个功能都可以分别关闭 `set_process（false）` 和 `set_physics_process（false）。`

## `_exit_tree()`

在从树中删除节点之前，它会运行 `_exit_tree（）` 函数。当您从父节点调用 `remove_child（）` 时，或者当您 `free（）` 或 `queue_free（）` 节点时，都会发生这种情况。与 `_ready（）` 类似，首先调用子节点的 `_exit_tree（）。`

节点在运行函数后会发出 `tree_exiting` 信号，一旦节点被完全移除，就会发出 `tree_exited` 信号。

退出树的节点的父节点将发出 `child_exiting_tree` 信号。