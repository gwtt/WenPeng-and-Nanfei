> 2025/11/13日做空洞骑士遇到的问题

首先第一个问题，遇到了CollisionPolygon2D在AnimationPlayer里面关键帧修改形状导致检测不到碰撞。

解决方案：只需要在AnimationPlayer或者AnimationTree的Callback Mode设置Process为Physics即可。

后来我用了替换方案，使用了CollisionShape2D，然后设置了ConvexPolygonShape2D，可以完成。但是出现了警告：
`Concave polygon is assigned to ConvexPolygonShape2D`

在 Godot 2D 物理里只有“静态”对象才能保持凹形，其余类型（KinematicBody2D、RigidBody2D、CharacterBody2D 等）都会被强行转换成凸形来做碰撞检测。等于Area2D不太能用

*凸多边形每个内角小于180度，凹多边形至少有一个内角大于180度*