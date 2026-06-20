# 模块十二：3D 入门 — 从 2D 出发（给 cheatsheet 作者的 plan）

> **目标学生**：2D topdown survivor 毕业生（已掌握 CharacterBody2D / Area2D / Timer / Camera2D / 信号 / process_mode / CanvasLayer）
> **课程定位**：最小 increment 进入 3D；不重讲 3D 概念，迁移已有经验
> **总课时**：3 节 × 2h（第四节可选）

---

## 核心迁移思路

| 你已经会的 2D | 3D 版本是这样 | 关键区别 |
|---|---|---|
| Node2D | Node3D | 多 Z 轴；rotation 是 Vector3 |
| Sprite2D | MeshInstance3D | 用网格模型而不是贴图 |
| Camera2D | Camera3D | 有 near/far/FOV；`look_at(target, up)` |
| CharacterBody2D | CharacterBody3D | `move_and_slide()` 参数略有不同 |
| Area2D | Area3D | 3D 碰撞体形状（BoxShape3D / SphereShape3D）|
| `area_entered` 信号 | `body_entered` 信号（Area3D 上）| 3D 场景用 body |
| `Input.get_vector(...)` | **完全一样** | 键盘映射不变 |
| `add_child(node)` | **完全一样** | 动态创建节点不变 |
| `queue_free()` | **完全一样** | 节点销毁不变 |

---

## 第 1 节：3D 节点 + 空场景跑起来

### 节点树
```
Main（Node3D）
├── MeshInstance3D（Cube，Position 0,0,0）  ← 代替 Sprite2D
├── DirectionalLight3D  ← 3D 必加：照亮场景
└── Camera3D（Position 0,5,10，Look At 0,0,0） ← 代替 Camera2D
```

### 重点 API

**MeshInstance3D**
- `Mesh` 属性 → `New BoxMesh` / `New SphereMesh`（选一个）
- 调颜色：Inspector → Material → 新建 `StandardMaterial3D` → `Albedo Color`

**DirectionalLight3D**
- `Shadow Enabled` = ON（打开阴影，更有立体感）
- `Light Color` 调暖白或日光色

**Camera3D**
- `Projection`：Perspective（透视）= 真实感；Orthogonal = 2D 感（本节用 Perspective）
- `FOV`：默认 75，越大越像鱼眼（新手建议 60）
- `Look At`：点击目标节点，让相机盯着它
- 快捷键：选中相机 → View → "Look From Here" 手动拖动相机到想要位置

### 相机绕立方体转（一行代码）

```gdscript
# Camera3D.gd
extends Camera3D

func _process(delta):
    look_at(Vector3.ZERO, Vector3.UP)
```

**解释**：
- `Vector3.ZERO` = 原点（0,0,0），立方体在原点
- `Vector3.UP` = Y 轴正方向（"头顶"向上），保证相机不翻转

### 验收
- [ ] F6 能跑起来，看见彩色立方体
- [ ] 能说出 `MeshInstance3D` 相当于 2D 的什么
- [ ] 能解释 `Camera3D.look_at()` 两个参数的意思

---

## 第 2 节：CharacterBody3D 角色移动

### 节点树
```
Main（Node3D）
├── Player（CharacterBody3D）  ← 核心
│    ├── MeshInstance3D（BoxMesh，缩小一点当角色）
│    └── CollisionShape3D（BoxShape3D）
├── Floor（StaticBody3D）  ← 地面
│    ├── MeshInstance3D（BoxMesh，Y 压扁）
│    └── CollisionShape3D
├── DirectionalLight3D
└── Camera3D（俯视 XZ 平面，Position 0,15,0，Look At Player）
```

### 移动代码（从 2D 迁移）

```gdscript
# Player.gd
extends CharacterBody3D

@export var speed: float = 10.0

func _physics_process(delta):
    # 2D 写法：velocity = Input.get_vector(...) * speed
    # 3D 写法：把 Vector2(X,Y) 变成 Vector3(X, 0, Y)
    # ——2D 的 Y 变成 3D 的 Z！
    var input_dir = Input.get_vector("left", "right", "up", "down")
    velocity = Vector3(input_dir.x, 0, input_dir.y) * speed
    move_and_slide()
```

**迁移口诀**：
> `Input.get_vector` 不变 → 结果 `Vector2(x, y)` → 变成 `Vector3(x, 0, y)` → Y 永远 = 0（锁定在 XZ 平面）= 2D topdown 的 3D 版本

### Camera follow（跟在 Player 后面）

```gdscript
# Main.gd
extends Node3D

@onready var camera: Camera3D = $Camera3D
@onready var player: CharacterBody3D = $Player

func _process(delta):
    camera.look_at(player.global_position, Vector3.UP)
```

### 验收
- [ ] WASD 能让 Player 在 XZ 平面移动
- [ ] 不会掉进 Floor 下面（CollisionShape3D 挡住）
- [ ] 能解释为什么 Y 固定为 0

---

## 第 3 节：子弹打靶（Area3D + 碰撞 + 信号）

### 节点树
```
Main（Node3D）
├── Player（CharacterBody3D）  ← 第 2 节已有
├── Target（StaticBody3D）  ← 新增：靶子
│    ├── MeshInstance3D（BoxMesh，红色）
│    └── CollisionShape3D
├── SpawnPoint（Marker3D）  ← 新增：子弹出生点（Player 前方）
├── BulletSpawner（Node3D）  ← 管理子弹
├── DirectionalLight3D
└── Camera3D
```

### Bullet.tscn 场景

```
Bullet（Area3D）  ← 用 Area3D 而不是 RigidBody（更简单）
├── MeshInstance3D（Small BoxMesh 或 SphereMesh）
├── CollisionShape3D
└── VisibleOnScreenNotifier3D  ← 出了屏幕就销毁
```

```gdscript
# Bullet.gd
extends Area3D

@export var speed: float = 30.0

func _physics_process(delta):
    # 一直朝前（-Z）飞
    position += -transform.basis.z * speed * delta

func _on_visible_on_screen_notifier_3d_screen_exited():
    queue_free()  # 出屏幕就销毁，不浪费性能
```

### 按空格发射

```gdscript
# Main.gd 或 Player.gd
@onready var bullet_scene = preload("res://bullet.tscn")
@onready var spawn_point: Marker3D = $SpawnPoint

func _input(event):
    if event.is_action_pressed("shoot"):
        var bullet = bullet_scene.instantiate()
        bullet.position = spawn_point.global_position
        bullet.transform.basis = spawn_point.transform.basis
        add_child(bullet)
```

### 靶子被击中（`body_entered` 信号）

```gdscript
# Target.gd
extends StaticBody3D

signal hit

func _on_body_entered(body):
    if body.is_in_group("bullet"):
        emit_signal("hit")
        body.queue_free()  # 子弹销毁
        queue_free()       # 靶子也销毁
```

> **注意**：`body_entered` 在 Area3D 上用，表示"有物理 body 进入我的范围"。
> `area_entered` 在另一个 Area3D 上用，表示"有 area 进入我的范围"。
> Bullet 是 Area3D，Target 是 StaticBody3D → Target 用 `body_entered`。

### 验收
- [ ] 按空格键，子弹从枪口位置飞出去
- [ ] 子弹碰到靶子，靶子消失
- [ ] 能解释 `body_entered` vs `area_entered`

---

## 第 4 节（可选）：2D survivor → 3D 移植

### 三件核心事（2D → 3D）

| 2D topdown survivor | 3D 移植 |
|---|---|
| Enemy spawn（Timer + add_child）| `add_child(enemy_instance)` 在 Timer timeout 里 |
| 子弹击中 Enemy → score++ | `Enemy.body_entered` → `Score.add_point()` |
| 升级菜单（CanvasLayer + process_mode=Always）| 3D 场景里 `add_child(CanvasLayer)`，逻辑完全一样 |

### 关键：CanvasLayer 在 3D 场景里叠加 UI

```
Main（3D 场景）
├── Player
├── Enemies
├── Bullets
└── UpgradeLayer（CanvasLayer）  ← 2D UI 可以叠加在任何 3D 场景上！
     └── CenterContainer → HBoxContainer → Button × 2
```

CanvasLayer 永远在最上层，跟下面的 3D 场景无关 —— 这是 module-10 "世界暂停，菜单继续"的经验直接复用。

### 验收
- [ ] 能玩：走 → 敌人生成 → 被打掉 → 分数增加
- [ ] 分数 > 10，升级菜单弹出来
- [ ] 能说出 CanvasLayer 在 3D 场景里的作用（跟 2D 场景完全一样）

---

## cheatsheet 写法提示（给朱古力）

本模块 cheatsheet 建议：

1. **不要**从零讲 3D 概念（学生会觉得在上一门新课）
2. **用 2D → 3D 迁移对照表** 作为主体框架
3. **代码块全部用 2D → 3D 并排对比**（学生扫码一看就懂差异在哪）
4. **只写关键 diff**，不抄完整代码（学生课后查，不是上课用）
5. **验收问题**每节结尾放 3 个（检查自己有没有跟上）
