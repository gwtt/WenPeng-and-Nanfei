https://github.com/nathanhoad/godot_dialogue_manager
# 基础语法

## 1.基本对话
```
This is some dialogue.
```

## 2.角色对话
```
Nathan: This is me talking.
```

## 3.BBCode格式化
除了 Godot RichTextLabel 的所有功能外，还提供了额外的标签：
- `[[选项1|选项2|选项3]]` - 在对话中随机选择一个选项（注意双方括号）
- `[wait=N]` - 暂停 N 秒后继续输出对话
- `[speed=N]` - 将默认打字速度乘以 N
- `[next=N]` - 等待 N 秒后自动继续到下一行对话，也可以使用 `[next=auto]` 让标签根据文本长度自动决定等待时间

## 4.多行对话
```
Nathan: I'll say this first.
Nathan: Then I'll say this line.
```
# 对话选项

## 1.基本选项
```
- This is a response
- This is a different response
- And this is the last one
```

## 2.嵌套对话
```
Nathan: How many projects have you started and not finished?
- Just a couple
	Nathan: That's not so bad.
- A lot
	Nathan: Maybe you should finish one before starting another one.
- I always finish my projects
	Nathan: That's great!
	Nathan: ...but how many is that?
	- A few
		Nathan: That's great!
	- I haven't actually started any
		Nathan: That's what I thought.
```

## 3.条件选项
选项可以包含条件来决定是否可选。在方括号中添加条件表达式：
```
- This is a normal response
- This is a conditional response [if SomeGlobal.some_property == true]
```

# 随机对话

## 1.随机行
使用 `%` 标记可以从多行中随机选择一行
```
Nathan: I will say this.
% Nathan: And then I might say this
% Nathan: Or maybe this
% Nathan: Or even this?
```

## 2.权重随机
使用 `%` 后跟数字来设置权重。例如，`%2` 表示该行被选中的概率是普通行的两倍
```
%3 Nathan: This line has a 60% chance of being picked
%2 Nathan: This line has a 40% chance of being picked
```

## 3.随机块
整个对话块也可以设为随机
如果选择了第一个随机项目，它将播放两条嵌套线。
```
%
	Nathan: This is the first block.
	Nathan: Still the first block.
% Nathan: This is the possible outcome.
```

# 变量使用

## 1.在对话中显示变量
```
Nathan: The value of some property is {{SomeGlobal.some_property}}.
```

## 2.动态角色名
```
{{SomeGlobal.some_character_name}}: My name was provided by the player.
```

## 3.标签
如果需要为对话行添加标签注释，用 `[#` 和 `]` 包裹，用逗号分隔
```
Nathan: [#happy, #surprised] Oh, Hello!
```
标签也可以包含值
```
Nathan: [#mood=happy] Oh, Hello!
```
对于这一行对话，`tags` 数组将是 `[“mood=happy”]`， `而 line.get_tag_value（“mood”）` 将返回 `“happy”`。

# 同时对话
如果您希望多个字符同时说话，可以使用并发行语法。在常规对话之后，任何同时要说的台词都可以以“| ”为前缀。
```
Nathan: This is a regular line of dialogue.
| Coco: And I'll say this line at the same time!
| Lilly: And I'll say this too!
```

# 标题和跳转

## 1.标题定义
标题是对话中的标记点，可以作为起点或跳转目标。标题以 `~` 开头
```
~ this_is_a_title
```

## 2.跳转
使用 `=>` 前缀跳转到指定标题
```
=> this_is_a_title
```

## 3.结束对话
跳转到 `END` 可以结束当前对话流程
```
=> END
```

## 4.跳转并返回
使用 `=><` 前缀可以跳转到某个标题，执行完后返回跳转点继续

## 5.选项中的跳转
```
~ start
Nathan: Well?
- First one
- Another one => another_title
- Start again => start
=> END

~ another_title
Nathan: Another one?
=> END
```

## 6.表达式跳转
您可以使用表达式作为跳转指令。表达式需要解析为已知的标题名称，否则结果将出乎意料。
请**谨慎使用这些选项** ，因为对话编译器无法在编译时验证表达式值与任何标题匹配。
表达式跳转如下所示：

=> {{SomeGlobal.some_property}}

## 7.导入对话
例如，我们可以有一个 `snippets.dialogue` 文件：
```
~ banter
Nathan: Blah blah blah.
=> END
```
然后我们可以将其导入到另一个对话文件中，并从片段文件跳转到`戏谑`标题（请注意 `=><` 语法，表示在跳转的对话完成后返回此行）：
```
import "res://snippets.dialogue" as snippets

~ start
Nathan: The next line will be from the snippets file:
=>< snippets/banter
Nathan: That was some banter!
=> END
```

# 条件和状态修改

## 1.if/else 条件
```
if SomeGlobal.some_property >= 10
    Nathan: That property is greater than or equal to 10
elif SomeGlobal.some_other_property == "some value"
    Nathan: Or we might be in here.
else
    Nathan: If neither are true, I'll say this.
```

## 2.内联条件
```
Nathan: You have {{num_apples}} [if num_apples == 1]apple[else]apples[/if], nice!
```

## 3.match 语句
用来替代一些if/elif链
```
match SomeGlobal.some_property
    when 1
        Nathan: It is 1.
    when > 5
        Nathan: It is less than 5 (but not 1).
    else
        Nathan: It was something else.
```

## 4.while 循环
```
while SomeGlobal.some_property < 10
    Nathan: The property is still less than 10 - specifically, it is {{SomeGlobal.some_property}}.
    do SomeGlobal.some_property += 1
Nathan: Now, we can move on.
```

## 5.状态修改（Mutations）
可以使用“set”或“do”行影响状态
```
if SomeGlobal.has_met_nathan == false
    do SomeGlobal.animate("Nathan", "Wave")
    Nathan: Hi, I'm Nathan.
    set SomeGlobal.has_met_nathan = true
Nathan: What can I do for you?
- Tell me more about this dialogue editor
```
在上面的示例中，对话管理器希望一个名为 `SomeGlobal` 的全局变量实现具有签名 `func animate(string, string) -> void` 的方法

## 6.内联修改
Mutations也可以内联使用。当输入的对话到达文本中的该点时，将调用Mutations
```
Nathan: I'm not sure we've met before [do wave()]I'm Nathan.
Nathan: I can also emit signals[do SomeGlobal.some_signal.emit()] inline.
```

## 7.信号发射
例如，如果 `SomeGlobal` 有一个名为 `some_signal` 的信号，该信号具有单个字符串参数，则可以从对话中发出它，如下所示
```
do SomeGlobal.some_signal.emit("some argument")
```

## 8.状态快捷方式
如果您想将对状态的引用从 `SomeGlobal.some_property` 之类的缩短为仅 `some_property`，有两种方法可以做到这一点。
- 如果您在所有对话中使用相同的状态，则可以在 [“设置”](https://github.com/nathanhoad/godot_dialogue_manager/blob/main/docs/Settings.md) 中设置全局状态快捷指令。
- 或者，如果你想要每个对话文件有不同的快捷方式，你可以在对话文件的顶部添加一个 `using SomeGlobal` 子句（适用于你正在使用的任何自动加载）。

## 9.特殊变量
- `do wait（float）` - 等待`浮点`秒数（内联使用时无效）。
- `do debug（...）`- 将某些内容打印到“输出”窗口。
- 还有一个特殊的属性 `self`，你可以在对话中使用它来引用当前正在运行的 `DialogueResource`。