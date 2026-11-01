# 模块十二：3D 入门 — 从 2D 出发

> **目标学生**：2D topdown survivor 毕业生（已掌握 `CharacterBody2D` / `Area2D` / `Timer` / `Camera2D` / 信号 / `process_mode` / `CanvasLayer`）
> **课程定位**：**最小 increment 进入 3D**；不重讲 3D 概念，迁移已有经验
> **总课时**：3 节 × 2h（**必上**）+ 第 4 节 × 2h（**可选**）
> **配套 lesson plan**：`projects/godot-landing-ppt/3d-onboarding/outline.md`（老师用的内部文档）

---

## 为什么这节从 2D 进 3D，而不是从零讲 3D

3D 入门**不是**一门新课。Godot 4 的 3D 子系统和 2D 子系统**共享同一套核心概念**（节点、信号、_process、_physics_process、add_child、queue_free、preload、instantiate）。学生**不是**零基础；他们只是**第一次看 3D 视角的 Godot**。

**核心论点**：

> **3D 入门 = 把 2D 经验搬到 3D 节点上**。Node2D → Node3D、CharacterBody2D → CharacterBody3D、Area2D → Area3D。**逻辑、API、思维方式全部不变**；唯一变的是节点的"3D 版本"长什么样、能做什么。

**本模块做的事**：

1. 3 节 × 2h = **6 小时**（2D 入门用了 11 节 × 2h = 22 小时；3D 入门是"用 6 小时验证你已经会的技能可以迁移"）
2. 4 节 X 2h（可选）= **完整 3D 版 topdown survivor mini demo**

**本模块不做的事**：

- ❌ 不重讲 3D 数学（向量、矩阵、变换——前置物理课学过）
- ❌ 不重讲 3D 美术（建模、贴图、骨骼——下学期 / 进阶班的事）
- ❌ 不从零讲 Godot UI / 信号 / Timer——module-1~11 已经讲过

---

## 2D → 3D 节点对照表（**核心**）

| 你已经会的 2D | 3D 版本是这样 | 关键差异 | 用法示例 |
|---|---|---|---|
| `Node2D` | `Node3D` | 多一个 Z 轴；`position` 是 `Vector3`（x/y/z）| `position = Vector3(0, 0, 5)` |
| `Sprite2D` | `MeshInstance3D` | 用 `Mesh`（BoxMesh / SphereMesh）而不是贴图精灵 | `Inspector → Mesh → New BoxMesh` |
| `Camera2D` | `Camera3D` | 有 `FOV` / `near` / `far` / `Projection`（Perspective / Orthogonal）| `camera.look_at(target, Vector3.UP)` |
| `CharacterBody2D` | `CharacterBody3D` | `velocity` 变 `Vector3`；`move_and_slide()` 一样 | `velocity = Vector3(input.x, 0, input.y) * speed` |
| `Area2D` | `Area3D` | 信号从 `area_entered` 变成 **`body_entered`**（**关键差异**）| `_on_body_entered(body): if body.is_in_group("bullet")` |
| `CollisionShape2D` + `RectangleShape2D` | `CollisionShape3D` + `BoxShape3D` / `SphereShape3D` | **必须用 3D 形状**（2D 形状在 3D 场景里不工作）| `Shape → New BoxShape3D` |
| `Marker2D` | `Marker3D` | 几乎一样（3D 多了 `transform.basis` 旋转信息）| 用作子弹出生点 / 生成点 |
| `Timer` | `Timer` | **完全一样**——`wait_time` / `autostart` / `timeout` 信号 | `Timer` + `_on_spawn_timer_timeout()` |
| `CanvasLayer` | `CanvasLayer` | **完全一样**——永远在最上层，2D UI 叠加 3D 场景 | 升级菜单、HUD 都可以 |
| `Input.get_vector("left", "right", "up", "down")` | `Input.get_vector("left", "right", "up", "down")` | **完全一样**——键盘映射不变 | `var input_dir = Input.get_vector(...)` |
| `add_child(node)` | `add_child(node)` | **完全一样** | `add_child(bullet_instance)` |
| `queue_free()` | `queue_free()` | **完全一样** | `queue_free()` |
| `preload("res://x.tscn")` + `.instantiate()` | `preload("res://x.tscn")` + `.instantiate()` | **完全一样** | `var bullet = bullet_scene.instantiate()` |
| `position` / `global_position` | `position` / `global_position` | **完全一样**（3D 用 `Vector3`）| `player.global_position` |
| `signal xxx` + `connect` | `signal xxx` + `connect` | **完全一样** | `target.hit.connect(_on_target_hit)` |
| `_process(delta)` / `_physics_process(delta)` | `_process(delta)` / `_physics_process(delta)` | **完全一样** | `func _physics_process(delta):` |
| `process_mode = ALWAYS` | `process_mode = ALWAYS` | **完全一样** | module-10 暂停时菜单能用 |
| `transform.basis.x` / `.y` | `transform.basis.x` / `.y` / **`.z`** | 3D 多了 Z 轴 | 子弹朝前飞 = `-transform.basis.z` |
| — | `DirectionalLight3D` | **3D 必加**（没光 = 全黑）| 太阳光 / 平行光，`Shadow Enabled = ON` |
| — | `WorldEnvironment` | 天空颜色 / 雾效 | `Background Mode = Sky` |

**一句话总结**：

> **80% 的 2D 经验 = 3D 经验**。剩下 20% 是 Z 轴、3D 形状、body_entered、DirectionalLight3D。

---

## 每节 1-2 句要点 + 最小代码片段

### 第 1 节：3D 节点 + 轨道相机

**1 句要点**：把 `Sprite2D` 换成 `MeshInstance3D`，把 `Camera2D` 换成 `Camera3D`，就能从 2D 切到 3D。

**最小代码**（orbital camera 1 行）：

```gdscript
# Camera3D.gd
extends Camera3D

func _process(delta):
    look_at(Vector3.ZERO, Vector3.UP)
```

---

### 第 2 节：`CharacterBody3D` 移动

**1 句要点**：`Input.get_vector` 返回 `Vector2`；**把它升级成 `Vector3(x, 0, y)`**——Y 永远 = 0 = 2D topdown 在 3D 的等价。

**最小代码**（WASD 移动）：

```gdscript
# Player.gd
extends CharacterBody3D

@export var speed: float = 10.0

func _physics_process(delta):
    var input_dir = Input.get_vector("left", "right", "up", "down")
    velocity = Vector3(input_dir.x, 0, input_dir.y) * speed
    move_and_slide()
```

---

### 第 3 节：子弹 + `Area3D` 打靶

**1 句要点**：`Area2D` 变 `Area3D`，但**信号从 `area_entered` 变成 `body_entered`**（因为 Target 是 `StaticBody3D` 是 body 不是 area）。

**最小代码**（子弹朝前飞 + 击中销毁）：

```gdscript
# Bullet.gd
extends Area3D

var speed: float = 30.0

func _physics_process(delta):
    position += -transform.basis.z * speed * delta  # 朝 -Z 飞

func _on_body_entered(body):
    if body.is_in_group("bullet"):
        body.queue_free()
        queue_free()
```

---

### 第 4 节（可选）：2D survivor → 3D 移植

**1 句要点**：3D demo 和 2D survivor 的核心三件事**逻辑完全一样**——`Timer` spawn 敌人 / 子弹打中敌人 → 计分 / 升级菜单用 `CanvasLayer` 叠加 3D 场景。

**最小代码**（Timer spawn 敌人）：

```gdscript
# Main.gd
@onready var enemy_scene = preload("res://enemy.tscn")
@onready var player: CharacterBody3D = $Player

func _on_spawn_timer_timeout():
    var enemy = enemy_scene.instantiate()
    enemy.position = player.position + Vector3(randf_range(-5, 5), 0, randf_range(-5, 5))
    add_child(enemy)
```

---

## 老板 3 句话（学生上完 3D 入门后能做什么）

> **第 1 句**：**能在 3D 空间里造一个能玩的场景**——3D 立方体 + 3D 光 + 3D 相机，能跑起来看。
>
> **第 2 句**：**能写 3D 版的角色控制 + 子弹打靶**——`CharacterBody3D` 移动、`Area3D` 子弹、`body_entered` 击中，**2D 经验全部迁移过来**。
>
> **第 3 句**：**能把 2D topdown survivor 翻译成 3D 版本**——`Timer` spawn 敌人 / 计分 / `CanvasLayer` 升级菜单 / `process_mode = ALWAYS` 暂停时菜单能用，**3D 场景的逻辑跟 2D 完全一样**，只是节点类型从 `2D` 变 `3D`。

---

## cheatsheet 写法提示（给朱古力）

> 本节是**给作者看的 plan**，不是给学生看的 cheatsheet HTML。**作者**：朱古力（按本 plan 转 HTML 风格 cheatsheet，**6/18 规则——slide 不写讲稿**）。

1. **不要**从零讲 3D 概念（学生会觉得在上一门新课）
2. **用 2D → 3D 迁移对照表** 作为 cheatsheet 主体框架
3. **代码块全部用 2D → 3D 并排对比**（学生扫码一看就懂差异在哪）
4. **只写关键 diff**，不抄完整代码（学生课后查，不是上课用）
5. **验收问题**每节结尾放 3 个（检查自己有没有跟上）
6. **口诀**用 2D→3D 对照口诀（"Input.get_vector 不变 → Vector2 → Vector3(x, 0, y) → Y=0 = 2D topdown"）
