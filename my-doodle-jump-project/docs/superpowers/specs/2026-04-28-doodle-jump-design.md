# Doodle Jump 网页游戏 — 设计规格说明

**日期**：2026-04-28  
**状态**：已确认，待实施  
**技术栈**：单文件 HTML5 Canvas + Vanilla JavaScript，无外部依赖

---

## 1. 项目概述

实现一个涂鸦跳跃（Doodle Jump）风格的网页小游戏，所有代码放在单个 `index.html` 文件中。玩家控制一个像素风格的橙色小方块机器人（Claude Code 吉祥物风格）在无限向上滚动的平台世界中跳跃，分数随高度增加，游戏难度随分数缓慢递进。

---

## 2. 画面与布局

- **Canvas 尺寸**：420 × 680px，居中于页面
- **背景色**：`#faf6f0`（浅米黄，模拟纸张质感）
- **背景纹理**：每 20px 一条浅灰横线（透明度 15%），随 cameraY 滚动
- **字体**：`'Comic Sans MS', cursive`（手写感）
- **分数显示**：当前分数左上角，历史最高分右上角
- **难度指示**：右下角小字显示当前区域（如"区域 3/10"）

---

## 3. 代码模块结构

单个 `<script>` 标签内按以下 9 个模块顺序组织，每模块以注释分隔：

```
1. CONFIG            — 所有常量集中管理
2. AudioManager      — Web Audio API 合成音效
3. DifficultyManager — 根据 cameraY 输出当前难度参数
4. Platform          — 平台类（普通 / 移动 / 易碎）
5. Player            — 玩家类（物理 / 渲染 / 动画）
6. GameState         — 全局状态变量
7. Core Functions    — init / update / render / gameLoop
8. Input             — 键盘与鼠标事件监听
9. Bootstrap         — 启动入口
```

---

## 4. CONFIG 常量表

```javascript
GRAVITY          = 0.4      // 每帧速度增量
JUMP_FORCE       = -14      // 踩台弹跳初速度
MOVE_SPEED       = 6        // 玩家左右移动速度（px/帧）
CANVAS_W         = 420
CANVAS_H         = 680
PLATFORM_W       = 70
PLATFORM_H       = 14
PLATFORM_COUNT   = 12       // 初始平台数量
PLATFORM_MIN     = 10       // 屏幕内最少平台数
PLATFORM_MAX     = 14       // 屏幕内最多平台数
PLATFORM_REFILL  = 100      // 最高平台距顶部超过此值时补充
MAX_GAP_INITIAL  = 120      // 初始最大平台纵向间距（px）
MAX_GAP_FINAL    = 180      // 最终最大平台纵向间距（px）
DIFFICULTY_CAP   = 20000    // cameraY 达到此值时难度上限
CAMERA_TRIGGER   = 0.4      // 角色高于画面此比例时镜头跟进
```

---

## 5. 坐标系约定

- 所有对象（Platform、Player）存储**世界坐标**（不含 cameraY 偏移）
- 仅在 `render()` 阶段做转换：`screenY = worldY - cameraY`
- `Platform.draw(ctx, cameraY)` 和 `Player.draw(ctx, cameraY)` 均接收 cameraY 参数，内部自行计算 screenY
- 此约定确保逻辑层与渲染层完全分离，避免坐标错位 bug

---

## 6. 玩家（Player 类）

### 6.1 外形（像素风橙色小方块机器人）

- **身体**：圆角矩形，36 × 32px，颜色 `#d4522a`（橙红）
- **眼睛**：两个 5 × 5px 深色方块，跟随 `facingDirection` 水平微偏 ±2px
- **腿**：底部两个 6 × 8px 小矩形，颜色略深

### 6.2 物理

```
每帧（update 开始时）：
  prevBottomY = worldY + 16        // 先记录上一帧脚部位置（身体高 32px，脚部 = 中心 + 16）
  velocityY  += GRAVITY
  worldY     += velocityY
  currentBottomY = worldY + 16     // 更新后的脚部位置，供碰撞检测使用
  worldX     += inputX * MOVE_SPEED

边界环绕：
  if (worldX < -18)             worldX = CANVAS_W + 18
  if (worldX > CANVAS_W + 18)  worldX = -18
```

### 6.3 压缩拉伸动画

| 状态 | scaleX | scaleY | 触发 |
|------|--------|--------|------|
| 正常飞行 | 1.0 | 1.0 | — |
| 跳跃拉伸 | 0.8 | 1.3 | velocityY < -8 |
| 落地压扁 | 1.3 | 0.7 | 碰撞弹跳瞬间 |
| 回弹速度 | 每帧 ±0.15 线性回弹至 1.0 |

实现：`ctx.save()` → `ctx.translate(cx, cy)` → `ctx.scale(scaleX, scaleY)` → 以原点绘制 → `ctx.restore()`

### 6.4 游戏结束判定

`player.worldY - cameraY > CANVAS_H + 50`（50px 缓冲防止误触发）

---

## 7. 平台（Platform 类）

### 7.1 三种类型

| 类型 | 颜色 | 行为 | 碰撞后 |
|------|------|------|--------|
| `'normal'` | `#5a9e5a` 绿色 | 静止 | 正常弹跳 |
| `'moving'` | `#5a7ec8` 蓝色 | 左右来回，碰边界反向 | 正常弹跳 |
| `'breakable'` | `#e8833a` 橙色 | 静止 | 触发消失动画，停止碰撞 |

### 7.2 生成规则

**初始生成**：
- 在 worldY ∈ [0, CANVAS_H] 内均匀分布 12 个平台
- 最底部一个强制为 `normal` 类型，位于玩家初始脚部正下方

**补充生成**：
- 检测时机：每帧检查最高平台的 worldY
- 条件：`minPlatformY > cameraY + PLATFORM_REFILL`
- 动作：在 `cameraY - 50` 处随机 X 位置生成一个新平台，直至数量达到 PLATFORM_MAX

**销毁条件**：`platform.worldY > cameraY + CANVAS_H`

**纵向间距保证**：
- 理论最大跳跃高度 = 14² / (2 × 0.4) = **245px**
- 生成时强制相邻平台纵向间距 ≤ `DifficultyManager.maxGap`（初始 120px，上限 180px）
- 安全余量 ≥ 65px，确保任何难度阶段玩家都能跳到下一个平台

### 7.3 碰撞检测（防穿透）

仅当以下 4 个条件全部满足时触发弹跳：

```
1. player.velocityY > 0                       （下落中）
2. player.prevBottomY <= platform.worldY      （上一帧脚部在平台上方）
3. player.currentBottomY >= platform.worldY  （当前帧脚部到达平台顶面）
4. 水平范围重叠：player.x - 18 < platform.right
                && player.x + 18 > platform.left
```

### 7.4 易碎平台消失动画

```
踩碎瞬间：state = 'dying', alpha = 1.0
每帧：    alpha -= 0.05
渲染：    ctx.globalAlpha = this.alpha
删除条件：alpha <= 0
碰撞检测：state === 'dying' 时跳过
```

---

## 8. 难度系统（DifficultyManager）

以 `cameraY` 为基准线性插值，`cameraY = DIFFICULTY_CAP(20000)` 时达到上限：

```javascript
t = Math.min(cameraY / DIFFICULTY_CAP, 1)

normalRatio    = 0.70 → 0.35   lerp(0.70, 0.35, t)
movingRatio    = 0.20 → 0.40   lerp(0.20, 0.40, t)
breakableRatio = 0.10 → 0.25   lerp(0.10, 0.25, t)
maxGap         = 120  → 180    lerp(120,  180,  t)
movingSpeed    = 2.0  → 5.0    lerp(2.0,  5.0,  t)
```

难度区域显示：`Math.floor(t * 10) + 1`（共 10 个区域，显示在右下角）

---

## 9. 摄像机系统

```javascript
// 每帧 update 中执行
if (player.worldY < cameraY + CANVAS_H * CAMERA_TRIGGER) {
  cameraY = player.worldY - CANVAS_H * CAMERA_TRIGGER
}

// 分数（只增不减）
score = Math.max(score, Math.floor(cameraY / 50))
```

---

## 10. 音效系统（AudioManager）

Web Audio API 纯合成，无外部文件。AudioContext 在首次用户交互时懒加载。

| 方法 | 音效 | 参数 |
|------|------|------|
| `playJump()` | 短促上扬 | 正弦波 300→600Hz，时长 0.12s |
| `playBreak()` | 碎裂短噪 | 噪音波形，快速衰减，时长 0.15s |
| `playGameOver()` | 下沉音效 | 正弦波 400→150Hz，时长 0.4s |
| `playMilestone()` | 清脆提示 | 正弦波 800Hz，时长 0.1s，每 100 分触发 |

---

## 11. 游戏状态管理

### 11.1 三个状态

**`'start'`（开始界面）**
- 显示：游戏标题（sin 函数驱动轻微浮动）、操作说明、历史最高分、静态平台示意
- 转换：空格键 / 鼠标点击 → `'playing'`，同时调用 `init()`

**`'playing'`（游戏进行中）**
- 主循环：update → render，实时显示分数与难度区域
- 转换：玩家坠出底部 → `'gameover'`

**`'gameover'`（游戏结束）**
- 显示：半透明黑色遮罩、"Game Over"、本局分数、历史最高分、重开提示
- 写入：`localStorage.setItem('doodleHighScore', highScore)`
- 转换：空格键 / 鼠标点击 → `'playing'`，调用 `init()` 重置

### 11.2 localStorage

- Key：`'doodleHighScore'`
- 读取时机：程序启动时
- 写入时机：进入 `gameover` 状态时

---

## 12. 视觉细节清单

| 效果 | 实现方式 |
|------|----------|
| 纸张横线纹理 | 每帧绘制，间距 20px，颜色 `rgba(0,0,0,0.04)`，随 cameraY mod 20 偏移 |
| 分数放大动画 | `scoreScale` 分数变化时设为 1.4，每帧 × 0.9 衰减至 1.0 |
| 易碎平台渐隐 | `alpha` 从 1.0 每帧 -0.05 至 0，过程中继续绘制但不碰撞 |
| 标题浮动动画 | `titleY = baseY + Math.sin(frame * 0.03) * 6` |
| 玩家压缩拉伸 | 见第 6.3 节 |

---

## 13. 输入事件

| 事件 | 操作 |
|------|------|
| `ArrowLeft` / `A` keydown | `inputX = -1` |
| `ArrowRight` / `D` keydown | `inputX = 1` |
| `ArrowLeft` / `A` / `ArrowRight` / `D` keyup | `inputX = 0` |
| `Space` keydown | 状态切换（start/gameover → playing） |
| `click` on canvas | 状态切换（start/gameover → playing） |

---

## 14. 产出物

- `index.html`：单文件，可直接双击用浏览器打开游玩
- 无需服务器，无需网络连接，无外部依赖
