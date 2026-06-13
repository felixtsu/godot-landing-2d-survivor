# 升级选择 Modal — 技术方案

> 给浩然/一曦（或蜗牛老师）看的"思路蓝图"。
> **不写完整 .gd 脚本** —— 学员拿到后自己手写。
> 时间：2026-06-13，方小虾

---

## 1. 节点结构（一棵树）

```
UpgradeModal (CanvasLayer)              ← 跟 L4 杀敌计分同一个 CanvasLayer 概念
  process_mode = PROCESS_MODE_ALWAYS    ← 关键！否则弹窗自己也冻住
  │
  ├── Dim (ColorRect)                   ← 半透明黑遮罩
  │     anchor_preset: full rect
  │     color: rgba(0, 0, 0, 0.6)
  │
  └── Popup (PanelContainer)            ← 中心弹窗容器
        anchor_preset: CENTER
        │
        └── VBox (VBoxContainer)        ← 自动垂直布局
              │
              ├── Title (Label)          ← "🎉 升级！选一个 buff"
              │
              ├── BuffA_Button (Button)
              │     text: "加速弹\n+30% 子弹速度"
              │
              ├── BuffB_Button (Button)
              │     text: "多发弹\n+1 子弹（同时射 2 发）"
              │
              └── (可选) SkipButton (Button)
                    text: "下次再说"
```

**为什么用这些节点**：

| 节点 | 选择理由 | 学员 L1-L5 学过吗 |
|------|----------|-----------------|
| **CanvasLayer** | 跟 module-8 计分 UI 同一个概念；不随相机移动，永远在最上层 | ✅ L4 |
| **ColorRect** | 最轻量的"遮罩"实现，单色填充一行搞定 | ⚠️ 简单 |
| **PanelContainer** | 自带 padding + 子节点布局，比裸 Panel 省事 | ⚠️ 简单 |
| **VBoxContainer** | module-7 没用过，但 L2 的 Player 节点树用过 child 结构；自动堆叠 | ⚠️ 简单 |
| **Button** | L5 Missile 的触发用 Button，学员熟 | ✅ L5 |

---

## 2. Freeze 游戏：怎么让游戏暂停

**核心 API**：`get_tree().paused = true / false`

**原理**（学员要理解的最重要一点）：

- `paused = true` 时，**所有节点**的 `_process()` / `_physics_process()` / `Timer` 全部停
- 视觉上 → 玩家 / 敌人 / 子弹定格在原地
- 但**信号仍然能触发**（`signal.connect()` 不受 paused 影响）
- **Input 仍然能收到**（`Input.is_action_pressed()` 不受 paused 影响）

**关键设置 —— 弹窗自己必须 `process_mode = ALWAYS`**：

```gdscript
# UpgradeModal.gd 的 _ready() 里
func _ready():
    process_mode = Node.PROCESS_MODE_ALWAYS
```

或者在 Inspector 选中节点 → Process Mode → Always。

> **不设会怎样**：弹窗自己也冻住 → Button 不能点击、Label 不更新、动画停。

**`process_mode` 的所有选项**（Godot 4 enum）：

| 值 | 含义 | 用途 |
|----|------|------|
| INHERIT (0) | 跟随父节点（默认） | 普通子节点 |
| PAUSABLE (1) | 跟全局 paused 走 | 游戏逻辑节点 |
| WHEN_PAUSED (2) | 只在 paused 时跑 | 暂停菜单 |
| ALWAYS (3) | 永远跑，不受 paused 影响 | **弹窗** |
| DISABLED (4) | 永远不跑 | 临时禁用 |

---

## 3. 状态机

学员在 Main.gd 里维护升级状态：

```
IDLE
  ↓ 玩家打怪，kill_count++
  ↓ 检测到升级
LEVEL_UP
  ↓
  ├── get_tree().paused = true
  ├── 实例化 UpgradeModal
  ├── modal.buff_pool = [buff1, buff2]   ← 抽卡
  ├── modal.buff_selected.connect(_on_buff_selected)
  └── add_child(modal)
        ↓ 等学员点按钮
BUFF_CHOSEN
  ↓
  ├── apply_buff(id)                    ← 应用 buff
  ├── get_tree().paused = false
  └── modal.queue_free()
        ↓
IDLE (新状态)
```

**伪代码**（学员自己写时实现）：

```gdscript
# Main.gd
@export var upgrade_modal_scene: PackedScene
const KILL_PER_LEVEL: int = 5

var kill_count: int = 0
var current_level: int = 1
var is_upgrading: bool = false

# 在 Bullet._on_area_entered 里，消灭敌人时调用
func add_kill():
    kill_count += 1
    if not is_upgrading and kill_count >= KILL_PER_LEVEL * current_level:
        _on_level_up()

func _on_level_up():
    is_upgrading = true
    get_tree().paused = true
    var modal = upgrade_modal_scene.instantiate()
    modal.buff_pool = _get_two_random_buffs()
    modal.buff_selected.connect(_on_buff_selected)
    add_child(modal)

func _on_buff_selected(buff_id: String):
    apply_buff(buff_id)
    current_level += 1
    is_upgrading = false
    get_tree().paused = false
```

---

## 4. Buff 数据：4-6 个静态定义

```gdscript
# upgrade_data.gd（可以是独立脚本，也可以是 Main.gd 里的常量）
const BUFF_POOL = [
    {
        "id": "faster_bullets",
        "name": "加速弹",
        "desc": "子弹速度 +30%"
    },
    {
        "id": "more_bullets",
        "name": "多发弹",
        "desc": "子弹数量 +1（同时射 2 发）"
    },
    {
        "id": "bigger_player",
        "name": "壮实",
        "desc": "玩家碰撞体 +20%（被碰到容错更大）"
    },
    {
        "id": "faster_player",
        "name": "神速",
        "desc": "玩家移动速度 +20%"
    },
    {
        "id": "explode_on_hit",
        "name": "爆炸弹",
        "desc": "子弹命中时在原地生成小爆炸"
    }
]
```

**抽 2 个不重复的 buff**：

```gdscript
func _get_two_random_buffs() -> Array:
    var pool = BUFF_POOL.duplicate()  # 复制一份，不打乱原数组
    pool.shuffle()                    # 随机洗牌
    return [pool[0], pool[1]]         # 取前 2
```

> **为什么不直接 random 2 次**：可能抽到同一个 buff。

---

## 5. 应用 buff：match 分发

```gdscript
# Player.gd 里加一个 apply_buff 方法
func apply_buff(buff_id: String):
    match buff_id:
        "faster_bullets":
            bullet_speed *= 1.3
        "more_bullets":
            bullet_count += 1
        "bigger_player":
            $CollisionShape2D.scale *= 1.2
        "faster_player":
            speed *= 1.2
        "explode_on_hit":
            # 这里只是标记状态，真正实现要 Bullet.gd 配合
            has_explode_buff = true
```

**注意**：
- `bullet_speed` / `bullet_count` / `speed` 都要在 Player.gd 里**先有 var 定义**
- `bullet_count` 改后，Player 发射子弹的循环要用 `for i in bullet_count:` 而不是写死 1
- "爆炸弹" 涉及 Bullet.gd 配合 —— 复杂 buff，学员可以选做

---

## 6. Modal 自身 —— UpgradeModal.gd 骨架

```gdscript
extends CanvasLayer

var buff_pool: Array = []  # 由 Main 在 instantiate 时注入
signal buff_selected(buff_id: String)

func _ready():
    # 1. 设置 process_mode（最关键的一行）
    process_mode = Node.PROCESS_MODE_ALWAYS

    # 2. 设置两个按钮的文本
    _setup_button($Popup/VBox/BuffA_Button, buff_pool[0])
    _setup_button($Popup/VBox/BuffB_Button, buff_pool[1])

    # 3. 连信号
    $Popup/VBox/BuffA_Button.pressed.connect(_on_a_pressed)
    $Popup/VBox/BuffB_Button.pressed.connect(_on_b_pressed)

func _setup_button(btn: Button, buff: Dictionary):
    btn.text = "%s\n%s" % [buff.name, buff.desc]

func _on_a_pressed():
    _select(buff_pool[0].id)

func _on_b_pressed():
    _select(buff_pool[1].id)

func _select(buff_id: String):
    buff_selected.emit(buff_id)
    queue_free()
```

**节点路径依赖** `$Popup/VBox/BuffA_Button`：
- 学员用 `@onready var` 更稳（避免路径打错）
- `get_node("Popup/VBox/BuffA_Button")` 也可以

---

## 7. 完整数据流（一次升级的 7 步）

```
1. Player 打死 Enemy
   ↓ Enemy.queue_free()
   ↓ Bullet.gd: 调用 get_node("/root/Main").add_kill()
2. Main.kill_count += 1
3. Main 检测 kill_count >= KILL_PER_LEVEL * level
4. Main._on_level_up()
   ├── get_tree().paused = true
   ├── 实例化 UpgradeModal
   ├── modal.buff_pool = [buff1, buff2]
   ├── modal.buff_selected.connect(_on_buff_selected)
   └── add_child(modal)
5. UpgradeModal._ready()
   ├── process_mode = ALWAYS
   ├── 渲染：半透明遮罩 + 中心弹窗 + 2 个按钮
   └── 等待玩家点
6. 玩家点 BuffA_Button
   ├── UpgradeModal._on_a_pressed()
   ├── emit signal "buff_selected" with id
   └── queue_free()
7. Main._on_buff_selected(buff_id)
   ├── Player.apply_buff(buff_id)
   ├── current_level += 1
   ├── is_upgrading = false
   └── get_tree().paused = false
   ↓
   玩家继续打怪，子弹/敌人/计时器全部恢复
```

---

## 8. 关键陷阱（要提醒学员）

| # | 陷阱 | 现象 | 解决 |
|---|------|------|------|
| 1 | **弹窗 process_mode 没设 ALWAYS** | 弹窗自己也冻住，按钮点不动 | `_ready()` 里设 `process_mode = Node.PROCESS_MODE_ALWAYS` |
| 2 | **抽 2 个 buff 抽到重复** | 两个按钮显示一样的 buff | 用 `duplicate() + shuffle() + 取前 2` |
| 3 | **buff 应用在 unpause 之后** | 玩家"看到 buff 还没生效"就恢复 | `apply_buff()` 必须在 `paused = false` **之前** |
| 4 | **bullet_count 改了但发射代码没改** | 变量 +1 但子弹还是只射 1 发 | 发射代码用 `for i in bullet_count:` 循环 |
| 5 | **collision scale 改了但 player image 没改** | 视觉不匹配，看着奇怪 | 学员自由选择要不要同步调 Sprite2D.scale |
| 6 | **match 漏 case** | 报 "no case matches" 错误 | 加通配 `_:` 或把所有 buff id 都列上 |
| 7 | **bullet 已飞出屏幕才升级** | bullet 还在飞但游戏暂停，bullet 视觉定格 | **这是想要的** —— 不要修 |
| 8 | **upgrade 检测跟 score 检测耦合** | kill_count 和 score 一起涨但用同一个 add_score() | 学员可分可合；建议分开写 add_kill() 和 add_score() |

---

## 9. 跟 L1-L5 概念的映射（学员复用）

| 方案里的概念 | 学员学过的课 | 复用方式 |
|------|------|------|
| CanvasLayer | L4 module-8 | 弹窗根节点 |
| PackedScene + instantiate | L2 module-3 | 升级 modal 场景 |
| signal connect / emit | L2 module-4 + L4 | buff_selected 回调 |
| queue_free | L2 module-4 | 关闭弹窗 |
| @export 字段 | L2 module-3 | upgrade_modal_scene 拖进 Inspector |
| @onready var | L2 module-3 | 引用 modal 子节点 |
| match | gdscript 基础 | apply_buff 分发 |
| Array.shuffle() | gdscript 基础 | 抽 buff |
| **新概念：process_mode** | **L5 没教** | 弹窗 ALWAYS |
| **新概念：get_tree().paused** | **L5 没教** | 全局暂停 |

**只引入 2 个新概念**：
- `process_mode = 3 (ALWAYS)` — 1 行属性
- `get_tree().paused = true/false` — 1 行 toggle

这两个其实非常简单，但**语义很关键**（不只是写代码，是理解"暂停是暂停谁"）。

---

## 10. 给学员的实现顺序（4 步，每步 F5 验证）

| 步骤 | 目标 | 验证方法 | 学到 |
|------|------|----------|------|
| **Step 1** | 按 P 键暂停/恢复 | 按 P → 玩家/敌人/子弹定格 → 再按 P → 恢复 | `get_tree().paused` 的全局效果 |
| **Step 2** | 按 P 弹 modal | 弹窗显示"Hello 升级！"，modal 上有按钮能关掉 | `process_mode = ALWAYS` 必要 |
| **Step 3** | modal 显示 2 个 buff | 抽卡算法工作，两个按钮显示不同的 buff 名字 | `Array.shuffle()` + 不重复 |
| **Step 4** | 点 buff → 应用 → 玩家真变快 | 玩家明显比之前快 / 子弹变多 | signal + apply_buff 完整链路 |

**每步独立可验证**，符合"建构式教学"节奏（跟 L1-L5 一脉相承）。

---

## 11. 进阶挑战（浩然/一曦 做完基本版后）

- **进阶 1**：3 选 1（Hades 风格）—— 抽 3 个，Modal 加一个 Button
- **进阶 2**：buff 互斥 —— "壮实"和"神速"不能同时选（用 `disabled = true`）
- **进阶 3**：buff 视觉化 —— 玩家身上显示"已选 buff 图标"（CanvasLayer + 多 Button）
- **进阶 4**：权重抽取 —— "加速弹"出现概率 5%，"壮实"出现概率 30%
- **进阶 5**：稀有度颜色 —— 蓝色普通 / 紫色稀有 / 橙色传说（按权重映射颜色）

---

## 12. 给老师的话

- **这是一个完整的"机制示范"** —— 学员做到 Step 4 就拥有了"幸存者"的核心循环
- **不需要在课堂上 demo 给全员** —— 让浩然/一曦自己摸索，**他们现在主动 flow 最强**
- **得宸/丰铭**还在 L2-L3 补基础，**不需要拉他们进这个项目**
- **手工写 4 步** = 4 节 1v1 课的体量
- 后续可以做成 module-10 cheatsheet（但要等浩然/一曦先自己趟一遍再写）

---

_2026-06-13 方小虾设计，参考 challenges.html 改造经验 + 浩然/一曦真实学习记录_
