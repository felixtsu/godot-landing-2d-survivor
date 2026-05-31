# SPEC.md — Godot 战斗系统实践手册

## 1. Concept & Vision

面向零基础学生的 Godot 2D 游戏开发实践手册网站。在教学现场，学生遇到问题时先查阅 FAQ 页面自救，减轻老师和答疑压力。风格简洁现代，卡片式布局，重点信息突出，配色活泼但不幼稚。

## 2. Design Language

沿用 bloom-2601 的 VI 体系，保持统一的教学品牌感。

### 色彩
| Token | Hex | 用途 |
|---|---|---|
| `--primary` | `#F3A21C` | 标题、强调、按钮 |
| `--secondary` | `#72ACC8` | 次级强调、标签 |
| `--accent` | `#667eea` | 装饰性边框、图标背景 |
| `--bg` | `#fafafa` | 页面背景 |
| `--card-bg` | `#ffffff` | 卡片背景 |
| `--text` | `#333333` | 正文 |
| `--text-muted` | `#666666` | 辅助文字 |

### 字体
- 正文：`'Segoe UI', Tahoma, Geneva, Verdana, sans-serif`
- 代码：`'Consolas', 'Monaco', 'Courier New', monospace`

### 动效
- 卡片 hover：`translateY(-5px)` + shadow 增强，300ms ease
- 页面载入：卡片依次 fadeInUp 动画，间隔 0.1s
- 无复杂交互动效，静态展示为主

### 空间
- 容器最大宽度：900px（内容页）/ 1200px（首页）
- 卡片圆角：15px
- 内边距：25–40px
- 区块间距：40px

## 3. Layout & Structure

### 页面结构

```
index.html          — 入口页（8个模块 + Challenges 附录的导航卡片 + 老师注记入口）
module-1.html       — 模块1：敌人的原型建模与动态生成
module-2.html       — 模块2：敌人识别与最近目标追踪
module-3.html       — 模块3：子弹的感应建模与发射
module-4.html       — 模块4：碰撞判定与信号反馈
module-5.html       — 模块5：计时器与自动化循环
module-6.html       — 模块6：TileMapLayer 建造你的世界
module-7.html       — 模块7：Camera2D 让相机跟着玩家走
module-8.html       — 模块8：CanvasLayer 独立的 GUI 层
challenges.html     — 附录：Challenges（按模块分组的分级挑战题）
styles.css          — 共享样式表
```

### index.html 结构
- 顶部标题区：网站名称 + 副标题
- 模块导航区：8个模块卡片 + 1个 Challenges 入口卡片（共9个）
- 老师注记区：单独入口卡片（汇总的老师注意事项）

### 内容页结构
- 顶部返回导航栏（← 返回首页）
- 页面大标题（模块编号 + 标题）
- 内容卡片（Features / Node Tree / Logic 三个 section）

## 4. Features & Interactions

### 导航
- 首页点击模块卡片 → 进入对应内容页
- 首页点击 Challenges 卡片 → 进入 `challenges.html`
- 内容页顶部 ← 返回首页
- 内容页无复杂交互，纯展示

### 模块8内容增强要求
- 在 `module-8.html` 增加 **Camera2D 与 CanvasLayer 配合的常见坑点**，明确两者属于不同坐标空间。
- 增加可执行的排错清单（节点层级、分数更新调用、相机平滑误判等）。
- 保持教学口吻：一句话结论 + 可直接操作的检查步骤。

### challenges.html 页面要求
- 作为附录页独立存在，包含模块1-8分组挑战题。
- 每组挑战统一采用 ⭐ / ⭐⭐ / ⭐⭐⭐ 难度分级。
- 页面需包含“使用方式”说明与“批改建议”区块，方便课堂与课后作业复用。

### 代码块
- `<pre><code>` 区域，背景 `#f5f5f5`，圆角 10px，左边框 4px accent 色
- GDScript 语法不做高亮（纯展示），保持可读性

### 响应式
- 移动端：单列布局，字号适当缩小
- 断点：768px

## 5. Component Inventory

### 模块卡片（首页）
- 白色背景，圆角 15px，阴影
- 左侧彩色竖条（按模块序号配色）
- hover 上浮 + 阴影增强
- 包含：模块编号、标题、简介

### 内容卡片
- 白色背景，圆角 15px
- 内含 section：Features、Node Tree、Logic
- section 标题带 emoji 图标

### 代码块
- 灰色背景，左边框 accent 色
- 等宽字体，padding 充足

### 老师注记卡片
- 橙色左边框，浅黄背景
- 特殊高亮样式

## 6. Technical Approach

- 纯静态 HTML + CSS，无 JS 依赖（无框架）
- 共享 `styles.css`
- 所有页面位于同一目录
- 编码：UTF-8
