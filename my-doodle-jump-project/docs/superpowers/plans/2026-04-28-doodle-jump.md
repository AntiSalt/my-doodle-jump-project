# Doodle Jump 网页游戏 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个可在浏览器中直接运行的涂鸦跳跃风格单文件网页游戏，玩家操控橙色像素小机器人在无限向上的平台世界中跳跃，难度随高度缓慢递增。

**Architecture:** 所有代码集中在单个 `index.html` 文件的 `<script>` 标签中，按 9 个模块顺序排列（CONFIG → AudioManager → DifficultyManager → Platform → Player → GameState → Core Functions → Input → Bootstrap）。渲染与逻辑通过世界坐标 + cameraY 偏移严格分离。

**Tech Stack:** HTML5 Canvas API、Vanilla JavaScript（ES6 class）、Web Audio API、localStorage。无任何外部依赖。

---

## 文件结构

| 文件 | 职责 |
|------|------|
| `index.html` | 唯一产出文件，包含全部 HTML / CSS / JS |

---

## Task 1：HTML 骨架 + Canvas 布局 + CONFIG 常量

**Files:**
- Create: `index.html`

- [ ] **Step 1: 创建 index.html，写入 HTML 骨架、Canvas 居中样式、CONFIG 对象**

```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <title>Doodle Jump</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #2c2c2c;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
    }
    canvas {
      display: block;
      border: 2px solid #888;
      border-radius: 4px;
    }
  </style>
</head>
<body>
  <canvas id="gameCanvas" width="420" height="680"></canvas>
  <script>
    'use strict';

    // =====================================================
    // 1. CONFIG — 所有常量集中管理
    // =====================================================
    const C = {
      GRAVITY:          0.4,
      JUMP_FORCE:      -14,
      MOVE_SPEED:       6,
      CANVAS_W:         420,
      CANVAS_H:         680,
      PLATFORM_W:       70,
      PLATFORM_H:       14,
      PLATFORM_COUNT:   12,
      PLATFORM_MIN:     10,
      PLATFORM_MAX:     14,
      PLATFORM_REFILL:  100,
      MAX_GAP_INITIAL:  120,
      MAX_GAP_FINAL:    180,
      DIFFICULTY_CAP:   20000,
      CAMERA_TRIGGER:   0.4,
    };

  </script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器中打开 index.html，验证**
  - 页面中央出现一个深色背景上的 420×680 浅色画布（此时画布为空白，只要不报错即可）
  - 打开浏览器开发者工具 Console，确认无报错

- [ ] **Step 3: git init 并提交**

```bash
cd "D:/Claude Code学習/my-doodle-jump-project"
git init
git add index.html
git commit -m "feat: html skeleton and CONFIG constants"
```

---

## Task 2：AudioManager — Web Audio API 音效合成

**Files:**
- Modify: `index.html`（在 CONFIG 注释块之后追加）

- [ ] **Step 1: 在 CONFIG 块之后追加 AudioManager 类**

```javascript
    // =====================================================
    // 2. AudioManager — Web Audio API 合成音效（懒加载）
    // =====================================================
    class AudioManager {
      constructor() {
        this._ctx = null;
      }

      _init() {
        if (!this._ctx) {
          this._ctx = new (window.AudioContext || window.webkitAudioContext)();
        }
      }

      _tone(freqStart, freqEnd, duration, gain = 0.25) {
        this._init();
        const ac = this._ctx;
        const osc = ac.createOscillator();
        const g   = ac.createGain();
        osc.connect(g);
        g.connect(ac.destination);
        osc.type = 'sine';
        osc.frequency.setValueAtTime(freqStart, ac.currentTime);
        osc.frequency.linearRampToValueAtTime(freqEnd, ac.currentTime + duration);
        g.gain.setValueAtTime(gain, ac.currentTime);
        g.gain.exponentialRampToValueAtTime(0.001, ac.currentTime + duration);
        osc.start(ac.currentTime);
        osc.stop(ac.currentTime + duration);
      }

      // 踩台弹跳：短促上扬音
      playJump() {
        this._tone(300, 600, 0.12);
      }

      // 踩碎平台：噪音碎裂感
      playBreak() {
        this._init();
        const ac = this._ctx;
        const bufSize = Math.floor(ac.sampleRate * 0.15);
        const buf  = ac.createBuffer(1, bufSize, ac.sampleRate);
        const data = buf.getChannelData(0);
        for (let i = 0; i < bufSize; i++) data[i] = Math.random() * 2 - 1;
        const src = ac.createBufferSource();
        const g   = ac.createGain();
        src.buffer = buf;
        src.connect(g);
        g.connect(ac.destination);
        g.gain.setValueAtTime(0.25, ac.currentTime);
        g.gain.exponentialRampToValueAtTime(0.001, ac.currentTime + 0.15);
        src.start();
      }

      // 游戏结束：下沉音效
      playGameOver() {
        this._tone(400, 150, 0.5, 0.3);
      }

      // 分数里程碑：清脆提示
      playMilestone() {
        this._tone(800, 800, 0.1, 0.2);
      }
    }

```

- [ ] **Step 2: 验证 AudioManager 可正常实例化**
  - 打开浏览器 Console，输入：
    ```javascript
    const a = new AudioManager();
    a.playJump();  // 首次交互后应听到短促上扬声
    ```
  - 确认无报错，且调用 `playJump()` 后能听到音效

- [ ] **Step 3: 提交**

```bash
git add index.html
git commit -m "feat: AudioManager with Web Audio API synthesis"
```

---

## Task 3：DifficultyManager — 难度线性插值系统

**Files:**
- Modify: `index.html`（在 AudioManager 块之后追加）

- [ ] **Step 1: 追加 DifficultyManager 类**

```javascript
    // =====================================================
    // 3. DifficultyManager — 根据 cameraY 输出难度参数
    // =====================================================
    class DifficultyManager {
      // cameraY 为负值（玩家向上移动时 cameraY 减小），取绝对值计算
      get(cameraY) {
        const t = Math.min(Math.abs(Math.min(cameraY, 0)) / C.DIFFICULTY_CAP, 1);
        const lerp = (a, b) => a + (b - a) * t;
        return {
          t,
          normalRatio:    lerp(0.70, 0.35),
          movingRatio:    lerp(0.20, 0.40),
          breakableRatio: lerp(0.10, 0.25),
          maxGap:         lerp(C.MAX_GAP_INITIAL, C.MAX_GAP_FINAL),
          movingSpeed:    lerp(2.0, 5.0),
          zone:           Math.floor(t * 10) + 1,
        };
      }
    }

```

- [ ] **Step 2: 在 Console 验证插值正确性**

```javascript
const d = new DifficultyManager();
// t=0 时（起始状态）
console.log(d.get(0));
// 期望：normalRatio≈0.70, movingRatio≈0.20, breakableRatio≈0.10, maxGap≈120, zone=1

// t=1 时（满难度）
console.log(d.get(-20000));
// 期望：normalRatio≈0.35, movingRatio≈0.40, breakableRatio≈0.25, maxGap≈180, zone=10

// 三种比例之和应恒为 1.0（验证）
const r = d.get(-10000);
console.log(r.normalRatio + r.movingRatio + r.breakableRatio); // 期望：1.0
```

- [ ] **Step 3: 提交**

```bash
git add index.html
git commit -m "feat: DifficultyManager with linear interpolation"
```

---

## Task 4：Platform 类 — 三种平台的渲染与碰撞

**Files:**
- Modify: `index.html`（在 DifficultyManager 块之后追加）

- [ ] **Step 1: 追加 Platform 类**

```javascript
    // =====================================================
    // 4. Platform — 普通 / 移动 / 易碎 平台类
    // =====================================================
    class Platform {
      constructor(x, worldY, type, speed = 2) {
        this.x      = x;
        this.worldY = worldY;
        this.w      = C.PLATFORM_W;
        this.h      = C.PLATFORM_H;
        this.type   = type;   // 'normal' | 'moving' | 'breakable'
        this.speed  = speed;
        this.dir    = 1;
        this.state  = 'alive'; // 'alive' | 'dying'
        this.alpha  = 1.0;
      }

      get left()  { return this.x; }
      get right() { return this.x + this.w; }
      get top()   { return this.worldY; }

      update() {
        if (this.type === 'moving' && this.state === 'alive') {
          this.x += this.speed * this.dir;
          if (this.x <= 0 || this.x + this.w >= C.CANVAS_W) {
            this.dir *= -1;
            // 防止卡边
            this.x = Math.max(0, Math.min(C.CANVAS_W - this.w, this.x));
          }
        }
        if (this.state === 'dying') {
          this.alpha -= 0.05;
        }
      }

      draw(ctx, cameraY) {
        const screenY = this.worldY - cameraY;
        if (screenY > C.CANVAS_H + 20 || screenY < -20) return;

        ctx.save();
        ctx.globalAlpha = this.alpha;

        const colors = {
          normal:    '#5a9e5a',
          moving:    '#5a7ec8',
          breakable: '#e8833a',
        };
        ctx.fillStyle = colors[this.type] || '#5a9e5a';

        // 圆角矩形（手动实现，兼容所有浏览器）
        const r = 6;
        const x = this.x, y = screenY, w = this.w, h = this.h;
        ctx.beginPath();
        ctx.moveTo(x + r, y);
        ctx.lineTo(x + w - r, y);
        ctx.arcTo(x + w, y,     x + w, y + r,     r);
        ctx.lineTo(x + w, y + h - r);
        ctx.arcTo(x + w, y + h, x + w - r, y + h, r);
        ctx.lineTo(x + r, y + h);
        ctx.arcTo(x,     y + h, x,     y + h - r, r);
        ctx.lineTo(x,    y + r);
        ctx.arcTo(x,     y,     x + r, y,          r);
        ctx.closePath();
        ctx.fill();

        // 移动平台加描边以区分
        if (this.type === 'moving') {
          ctx.strokeStyle = 'rgba(255,255,255,0.4)';
          ctx.lineWidth = 1.5;
          ctx.stroke();
        }

        ctx.restore();
      }

      // 碰撞检测（防穿透版，需传入 player）
      checkCollision(player) {
        if (this.state !== 'alive')          return false;
        if (player.velocityY <= 0)           return false;
        if (player.prevBottomY > this.worldY) return false;
        if (player.currentBottomY < this.worldY) return false;
        if (player.worldX + 18 <= this.left) return false;
        if (player.worldX - 18 >= this.right) return false;
        return true;
      }

      // 被踩后调用
      onLand() {
        if (this.type === 'breakable') {
          this.state = 'dying';
          this.alpha = 1.0;
        }
      }
    }

```

- [ ] **Step 2: 在 Console 临时验证 Platform 能渲染**

```javascript
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
ctx.fillStyle = '#faf6f0';
ctx.fillRect(0, 0, 420, 680);

// 各类型平台各画一个
const p1 = new Platform(50,  200, 'normal');
const p2 = new Platform(200, 350, 'moving', 3);
const p3 = new Platform(300, 500, 'breakable');
[p1, p2, p3].forEach(p => p.draw(ctx, 0));
// 期望：画布上出现绿色、蓝色、橙色三个圆角矩形
```

- [ ] **Step 3: 提交**

```bash
git add index.html
git commit -m "feat: Platform class with three types, draw and collision"
```

---

## Task 5：Player 类 — 像素机器人渲染与物理

**Files:**
- Modify: `index.html`（在 Platform 块之后追加）

- [ ] **Step 1: 追加 Player 类**

```javascript
    // =====================================================
    // 5. Player — 玩家类（物理 / 渲染 / 压缩拉伸动画）
    // =====================================================
    class Player {
      constructor(worldX, worldY) {
        this.worldX         = worldX;
        this.worldY         = worldY;
        this.velocityY      = 0;
        this.prevBottomY    = worldY + 16;
        this.currentBottomY = worldY + 16;
        this.scaleX         = 1.0;
        this.scaleY         = 1.0;
        this.facingDir      = 1;  // 1=右, -1=左
      }

      update(inputX) {
        // 记录上一帧脚部 Y（碰撞检测用）
        this.prevBottomY = this.worldY + 16;

        // 物理
        this.velocityY  += C.GRAVITY;
        this.worldY     += this.velocityY;
        this.currentBottomY = this.worldY + 16;

        // 水平移动
        if (inputX !== 0) this.facingDir = inputX;
        this.worldX += inputX * C.MOVE_SPEED;

        // 边界环绕
        if (this.worldX < -18)               this.worldX = C.CANVAS_W + 18;
        if (this.worldX > C.CANVAS_W + 18)   this.worldX = -18;

        // 压缩拉伸：向上飞行时拉伸，向下时恢复
        const targetScaleX = this.velocityY < -8 ? 0.8 : 1.0;
        const targetScaleY = this.velocityY < -8 ? 1.3 : 1.0;
        this.scaleX += (targetScaleX - this.scaleX) * 0.15;
        this.scaleY += (targetScaleY - this.scaleY) * 0.15;
        // 将 scale 钳位到合理范围，防止浮点漂移
        this.scaleX = Math.max(0.5, Math.min(1.5, this.scaleX));
        this.scaleY = Math.max(0.5, Math.min(1.5, this.scaleY));
      }

      // 触发弹跳（碰撞时调用）
      land() {
        this.velocityY = C.JUMP_FORCE;
        // 落地压扁
        this.scaleX = 1.3;
        this.scaleY = 0.7;
      }

      draw(ctx, cameraY) {
        const sx = Math.round(this.worldX);
        const sy = Math.round(this.worldY - cameraY);

        ctx.save();
        ctx.translate(sx, sy);
        ctx.scale(this.scaleX, this.scaleY);

        // --- 身体（36×32 圆角矩形，以中心为原点）---
        const bw = 36, bh = 32, br = 5;
        const bx = -bw / 2, by = -bh / 2;
        ctx.fillStyle = '#d4522a';
        ctx.beginPath();
        ctx.moveTo(bx + br, by);
        ctx.lineTo(bx + bw - br, by);
        ctx.arcTo(bx + bw, by,      bx + bw, by + br,      br);
        ctx.lineTo(bx + bw, by + bh - br);
        ctx.arcTo(bx + bw, by + bh, bx + bw - br, by + bh, br);
        ctx.lineTo(bx + br, by + bh);
        ctx.arcTo(bx,      by + bh, bx,      by + bh - br, br);
        ctx.lineTo(bx,     by + br);
        ctx.arcTo(bx,      by,      bx + br, by,            br);
        ctx.closePath();
        ctx.fill();

        // --- 高光（顶部浅色条纹，增加立体感）---
        ctx.fillStyle = 'rgba(255,255,255,0.18)';
        ctx.beginPath();
        ctx.moveTo(bx + br, by);
        ctx.lineTo(bx + bw - br, by);
        ctx.arcTo(bx + bw, by, bx + bw, by + br, br);
        ctx.lineTo(bx + bw, by + 10);
        ctx.lineTo(bx, by + 10);
        ctx.lineTo(bx, by + br);
        ctx.arcTo(bx, by, bx + br, by, br);
        ctx.closePath();
        ctx.fill();

        // --- 眼睛（随 facingDir 偏移 2px）---
        const eo = this.facingDir * 2;
        // 眼白
        ctx.fillStyle = '#fff8f0';
        ctx.fillRect(-12 + eo, -10, 7, 7);
        ctx.fillRect(5  + eo, -10, 7, 7);
        // 瞳孔
        ctx.fillStyle = '#1a0a00';
        ctx.fillRect(-11 + eo, -9,  5, 5);
        ctx.fillRect(6   + eo, -9,  5, 5);
        // 高光点
        ctx.fillStyle = '#ffffff';
        ctx.fillRect(-10 + eo, -8,  2, 2);
        ctx.fillRect(7   + eo, -8,  2, 2);

        // --- 腿（底部两个小矩形）---
        ctx.fillStyle = '#a03820';
        ctx.fillRect(-14, 15, 7, 9);
        ctx.fillRect(7,   15, 7, 9);
        // 脚掌
        ctx.fillStyle = '#7a2810';
        ctx.fillRect(-15, 22, 9,  3);
        ctx.fillRect(6,   22, 9,  3);

        ctx.restore();
      }
    }

```

- [ ] **Step 2: 在 Console 验证 Player 渲染**

```javascript
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
ctx.fillStyle = '#faf6f0';
ctx.fillRect(0, 0, 420, 680);

const p = new Player(210, 340);
p.draw(ctx, 0);
// 期望：画面中央出现橙红色像素小机器人，有眼睛和腿
// 正面时两眼居中，眼白+瞳孔+高光点清晰可见
```

- [ ] **Step 3: 提交**

```bash
git add index.html
git commit -m "feat: Player class with pixel robot drawing and physics"
```

---

## Task 6：GameState + init() — 全局状态与初始化

**Files:**
- Modify: `index.html`（在 Player 块之后追加）

- [ ] **Step 1: 追加 GameState 变量与 init 函数**

```javascript
    // =====================================================
    // 6. GameState — 全局状态变量
    // =====================================================
    const audio      = new AudioManager();
    const difficulty = new DifficultyManager();

    const canvas = document.getElementById('gameCanvas');
    const ctx    = canvas.getContext('2d');

    let gameState    = 'start';
    let platforms    = [];
    let player       = null;
    let cameraY      = 0;
    let score        = 0;
    let highScore    = parseInt(localStorage.getItem('doodleHighScore') || '0', 10);
    let inputX       = 0;
    let frame        = 0;
    let scoreScale   = 1.0;
    let lastMilestone = 0;

    // =====================================================
    // 7. Core Functions — init / spawnPlatform / update / render / gameLoop
    // =====================================================

    function pickType(diff) {
      const r = Math.random();
      if (r < diff.normalRatio) return 'normal';
      if (r < diff.normalRatio + diff.movingRatio) return 'moving';
      return 'breakable';
    }

    function init() {
      cameraY       = 0;
      score         = 0;
      frame         = 0;
      scoreScale    = 1.0;
      lastMilestone = 0;
      platforms     = [];

      const diff = difficulty.get(0);

      // 第一个平台固定在玩家脚下，保证游戏开始时不坠落
      const p0x = C.CANVAS_W / 2 - C.PLATFORM_W / 2;
      const p0y = C.CANVAS_H - 100;
      platforms.push(new Platform(p0x, p0y, 'normal'));

      // 向上均匀生成剩余 11 个平台
      let lastY = p0y;
      for (let i = 1; i < C.PLATFORM_COUNT; i++) {
        const gap  = 60 + Math.random() * (diff.maxGap - 60);
        const y    = lastY - gap;
        const x    = Math.random() * (C.CANVAS_W - C.PLATFORM_W);
        const type = pickType(diff);
        platforms.push(new Platform(x, y, type, diff.movingSpeed));
        lastY = y;
      }

      // 玩家初始位于第一个平台正上方，带初速弹起
      player = new Player(C.CANVAS_W / 2, p0y - 40);
      player.velocityY = C.JUMP_FORCE;
    }

```

- [ ] **Step 2: 在 Console 验证 init() 正确生成平台**

```javascript
init();
console.log('平台数量:', platforms.length);         // 期望：12
console.log('最低平台 Y:', Math.max(...platforms.map(p => p.worldY)));  // 期望：~580
console.log('最高平台 Y:', Math.min(...platforms.map(p => p.worldY)));  // 期望：负数或接近0
console.log('玩家初始 Y:', player.worldY);           // 期望：~540
console.log('玩家初速:', player.velocityY);          // 期望：-14
```

- [ ] **Step 3: 提交**

```bash
git add index.html
git commit -m "feat: GameState variables and init() platform generation"
```

---

## Task 7：游戏主循环 — update + render + gameLoop

**Files:**
- Modify: `index.html`（紧接 init 函数之后追加）

- [ ] **Step 1: 追加 spawnPlatform、update、三个 render 函数、gameLoop**

```javascript
    function spawnPlatform() {
      const diff = difficulty.get(cameraY);
      // 在当前所有平台中最高的那个再往上 gap 处生成
      const minY = platforms.reduce((m, p) => Math.min(m, p.worldY), Infinity);
      const gap  = 60 + Math.random() * (diff.maxGap - 60);
      const y    = minY - gap;
      const x    = Math.random() * (C.CANVAS_W - C.PLATFORM_W);
      platforms.push(new Platform(x, y, pickType(diff), diff.movingSpeed));
    }

    function update() {
      if (gameState !== 'playing') return;

      // 更新所有平台
      for (const p of platforms) p.update();

      // 更新玩家
      player.update(inputX);

      // 碰撞检测（只检测活跃平台）
      for (const p of platforms) {
        if (p.checkCollision(player)) {
          player.land();
          p.onLand();
          audio.playJump();
          if (p.type === 'breakable') audio.playBreak();
          break; // 同一帧只触发一次弹跳
        }
      }

      // 摄像机跟随（玩家进入上 40% 区域时）
      const triggerY = cameraY + C.CANVAS_H * C.CAMERA_TRIGGER;
      if (player.worldY < triggerY) {
        cameraY = player.worldY - C.CANVAS_H * C.CAMERA_TRIGGER;
      }

      // 分数（只增不减，基于 cameraY 负值的绝对值）
      const newScore = Math.max(score, Math.floor(Math.abs(Math.min(cameraY, 0)) / 50));
      if (newScore > score) {
        score = newScore;
        scoreScale = 1.4;
        // 每 100 分播放里程碑音
        if (Math.floor(score / 100) > Math.floor(lastMilestone / 100)) {
          audio.playMilestone();
          lastMilestone = score;
        }
      }
      if (scoreScale > 1.0) scoreScale = Math.max(1.0, scoreScale * 0.9);

      // 清理：删除死亡平台和已落出屏幕底部的平台
      platforms = platforms.filter(p => {
        if (p.state === 'dying' && p.alpha <= 0) return false;
        if (p.worldY > cameraY + C.CANVAS_H + 50)  return false;
        return true;
      });

      // 补充平台（最高平台离屏顶超过 PLATFORM_REFILL 时）
      const minY = platforms.reduce((m, p) => Math.min(m, p.worldY), Infinity);
      if (minY > cameraY + C.PLATFORM_REFILL) {
        while (platforms.length < C.PLATFORM_MAX) {
          spawnPlatform();
          // 防止死循环：重新检查 minY
          const newMin = platforms.reduce((m, p) => Math.min(m, p.worldY), Infinity);
          if (newMin <= cameraY + C.PLATFORM_REFILL) break;
        }
      }

      // 游戏结束判定
      if (player.worldY - cameraY > C.CANVAS_H + 50) {
        if (score > highScore) {
          highScore = score;
          localStorage.setItem('doodleHighScore', String(highScore));
        }
        audio.playGameOver();
        gameState = 'gameover';
      }
    }

    // --- 背景（纸张横线纹理）---
    function drawBackground() {
      ctx.fillStyle = '#faf6f0';
      ctx.fillRect(0, 0, C.CANVAS_W, C.CANVAS_H);

      ctx.strokeStyle = 'rgba(0,0,0,0.04)';
      ctx.lineWidth = 1;
      const offset = ((cameraY % 20) + 20) % 20;
      for (let y = offset; y < C.CANVAS_H; y += 20) {
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(C.CANVAS_W, y);
        ctx.stroke();
      }
    }

    // --- HUD（分数 / 最高分 / 区域）---
    function drawHUD() {
      ctx.save();

      // 当前分数（左上，带放大动画）
      ctx.save();
      ctx.translate(16, 28);
      ctx.scale(scoreScale, scoreScale);
      ctx.font      = "bold 18px 'Comic Sans MS', cursive";
      ctx.fillStyle = '#333333';
      ctx.textAlign = 'left';
      ctx.fillText(`分数: ${score}`, 0, 0);
      ctx.restore();

      // 最高分（右上）
      ctx.font      = "14px 'Comic Sans MS', cursive";
      ctx.fillStyle = '#666666';
      ctx.textAlign = 'right';
      ctx.fillText(`最高: ${highScore}`, C.CANVAS_W - 12, 24);

      // 区域指示（右下）
      const diff = difficulty.get(cameraY);
      ctx.font      = "12px 'Comic Sans MS', cursive";
      ctx.fillStyle = '#aaaaaa';
      ctx.fillText(`区域 ${diff.zone}/10`, C.CANVAS_W - 12, C.CANVAS_H - 12);

      ctx.restore();
    }

    // --- 开始界面 ---
    function renderStart() {
      drawBackground();

      // 静态示意平台
      const demos = [[50, 500], [190, 400], [300, 320], [80, 240]];
      const demoColors = ['#5a9e5a', '#5a7ec8', '#e8833a', '#5a9e5a'];
      demos.forEach(([x, y], i) => {
        ctx.fillStyle = demoColors[i];
        const r = 6, w = C.PLATFORM_W, h = C.PLATFORM_H;
        ctx.beginPath();
        ctx.moveTo(x + r, y);
        ctx.lineTo(x + w - r, y);
        ctx.arcTo(x + w, y,     x + w, y + r,     r);
        ctx.lineTo(x + w, y + h - r);
        ctx.arcTo(x + w, y + h, x + w - r, y + h, r);
        ctx.lineTo(x + r, y + h);
        ctx.arcTo(x,     y + h, x,     y + h - r, r);
        ctx.lineTo(x,    y + r);
        ctx.arcTo(x,     y,     x + r, y,          r);
        ctx.closePath();
        ctx.fill();
      });

      // 浮动标题（sin 波动）
      const titleY = 200 + Math.sin(frame * 0.03) * 6;
      ctx.textAlign  = 'center';
      ctx.fillStyle  = '#d4522a';
      ctx.font       = "bold 52px 'Comic Sans MS', cursive";
      ctx.fillText('Doodle', C.CANVAS_W / 2, titleY);
      ctx.fillText('Jump!',  C.CANVAS_W / 2, titleY + 60);

      // 最高分
      ctx.font      = "16px 'Comic Sans MS', cursive";
      ctx.fillStyle = '#666';
      ctx.fillText(`最高分: ${highScore}`, C.CANVAS_W / 2, titleY + 115);

      // 操作提示（呼吸闪烁）
      ctx.globalAlpha = 0.5 + 0.5 * Math.sin(frame * 0.06);
      ctx.font        = "17px 'Comic Sans MS', cursive";
      ctx.fillStyle   = '#333';
      ctx.fillText('按空格键或点击开始', C.CANVAS_W / 2, C.CANVAS_H - 70);
      ctx.globalAlpha = 1;
    }

    // --- 游戏进行中界面 ---
    function renderPlaying() {
      drawBackground();
      for (const p of platforms) p.draw(ctx, cameraY);
      player.draw(ctx, cameraY);
      drawHUD();
    }

    // --- 游戏结束界面 ---
    function renderGameOver() {
      renderPlaying();

      // 半透明遮罩
      ctx.fillStyle = 'rgba(0,0,0,0.52)';
      ctx.fillRect(0, 0, C.CANVAS_W, C.CANVAS_H);

      ctx.textAlign = 'center';

      ctx.font      = "bold 54px 'Comic Sans MS', cursive";
      ctx.fillStyle = '#ffffff';
      ctx.fillText('Game Over', C.CANVAS_W / 2, 260);

      ctx.font      = "22px 'Comic Sans MS', cursive";
      ctx.fillStyle = '#eeeeee';
      ctx.fillText(`分数: ${score}`,      C.CANVAS_W / 2, 320);
      ctx.fillText(`最高分: ${highScore}`, C.CANVAS_W / 2, 356);

      ctx.globalAlpha = 0.5 + 0.5 * Math.sin(frame * 0.06);
      ctx.font        = "16px 'Comic Sans MS', cursive";
      ctx.fillStyle   = '#cccccc';
      ctx.fillText('按空格键或点击重新开始', C.CANVAS_W / 2, 420);
      ctx.globalAlpha = 1;
    }

    function render() {
      if      (gameState === 'start')    renderStart();
      else if (gameState === 'playing')  renderPlaying();
      else if (gameState === 'gameover') renderGameOver();
    }

    function gameLoop() {
      frame++;
      update();
      render();
      requestAnimationFrame(gameLoop);
    }

```

- [ ] **Step 2: 临时在末尾调用 gameLoop() 验证（先不加 input）**

在 `</script>` 之前追加：
```javascript
    init();
    gameState = 'playing';
    gameLoop();
```

刷新浏览器，验证：
- 画布背景为浅米黄色，有淡淡横线纹理
- 平台（绿/蓝/橙色圆角矩形）可见
- 橙红色小机器人在屏幕上方弹跳
- 左上角显示"分数: 0"，右上角显示"最高: 0"
- 随着机器人上升，镜头跟随，分数增加
- 机器人最终坠落，切换到 gameover 状态，显示遮罩和结果

**若出现以下问题对应处理：**
- 机器人立即坠落不弹跳 → 检查 `init()` 中第一个平台 Y 是否在 `player.worldY + 16` 附近
- 屏幕变黑 → Console 查看报错，通常是 `platforms.reduce` 在空数组上失败

- [ ] **Step 3: 确认无误后移除临时调用（这两行将在 Task 9 正确加入）**

删除刚才临时追加的：
```javascript
    init();
    gameState = 'playing';
    gameLoop();
```

- [ ] **Step 4: 提交**

```bash
git add index.html
git commit -m "feat: core game loop with update/render/camera/scoring"
```

---

## Task 8：输入事件监听 + Bootstrap 启动

**Files:**
- Modify: `index.html`（在 gameLoop 函数之后追加）

- [ ] **Step 1: 追加 Input 监听与 Bootstrap**

```javascript
    // =====================================================
    // 8. Input — 键盘与 Canvas 点击事件
    // =====================================================
    document.addEventListener('keydown', e => {
      if (e.code === 'ArrowLeft'  || e.code === 'KeyA') inputX = -1;
      if (e.code === 'ArrowRight' || e.code === 'KeyD') inputX =  1;
      if (e.code === 'Space') {
        e.preventDefault(); // 防止页面滚动
        if (gameState === 'start' || gameState === 'gameover') {
          init();
          gameState = 'playing';
        }
      }
    });

    document.addEventListener('keyup', e => {
      if (['ArrowLeft','KeyA','ArrowRight','KeyD'].includes(e.code)) {
        inputX = 0;
      }
    });

    canvas.addEventListener('click', () => {
      if (gameState === 'start' || gameState === 'gameover') {
        init();
        gameState = 'playing';
      }
    });

    // =====================================================
    // 9. Bootstrap — 启动入口
    // =====================================================
    gameLoop();
```

- [ ] **Step 2: 刷新浏览器，完整功能验证清单**

  **开始界面：**
  - [ ] 画面显示浮动标题"Doodle Jump!"，标题上下微微飘动
  - [ ] 底部操作提示文字呼吸闪烁
  - [ ] 画面上有 4 个示意平台（绿、蓝、橙、绿）

  **游戏进行：**
  - [ ] 按空格键或点击画面，游戏开始
  - [ ] 按左右方向键（或 A/D），小机器人左右移动，眼睛方向随之改变
  - [ ] 小机器人自动在平台上弹跳，每次弹跳有音效
  - [ ] 踩橙色易碎平台后，平台渐渐消失，踩碎有音效
  - [ ] 小机器人向上弹跳时拉伸，落地瞬间压扁
  - [ ] 屏幕上方不断出现新平台，旧平台在底部消失
  - [ ] 左上角分数随上升增加（分数增加时有短暂放大效果）
  - [ ] 右下角显示"区域 X/10"
  - [ ] 从左边缘穿过去从右边缘出现（反之亦然）

  **游戏结束：**
  - [ ] 机器人坠出底部，画面变暗，显示"Game Over"、分数、最高分
  - [ ] 有低沉结束音效
  - [ ] 按空格或点击，重新开始
  - [ ] 刷新页面后历史最高分仍然保留（localStorage 验证）

- [ ] **Step 3: 提交**

```bash
git add index.html
git commit -m "feat: input handling and bootstrap, complete playable game"
```

---

## Task 9：最终集成验证与收尾

**Files:**
- Modify: `index.html`（若发现 bug 则修复）

- [ ] **Step 1: 边界场景验证**

  以下场景逐一手动测试：

  | 场景 | 操作 | 期望结果 |
  |------|------|----------|
  | 高速穿透 | 故意让角色从高处快速下落 | 不穿过平台，正常弹跳 |
  | 连续易碎 | 连续踩多个橙色平台 | 每个都消失，无碰撞残留 |
  | 长时间游戏 | 坚持到"区域 5/10" | 移动平台明显加速，蓝橙平台比例增加 |
  | 最高分更新 | 打出新纪录后刷新页面 | 开始界面显示新最高分 |
  | 键盘快速切换 | 快速左右交替按键 | 无卡顿，眼睛方向实时更新 |

- [ ] **Step 2: 确认文件为纯单文件可打开**

```bash
# 检查文件中是否有任何 http/外部 URL 引用
grep -n "http" "D:/Claude Code学習/my-doodle-jump-project/index.html"
# 期望：无任何输出（或只有 html lang 等非资源引用）
```

- [ ] **Step 3: 最终提交**

```bash
git add index.html
git commit -m "feat: doodle jump game complete - all features verified"
```

---

## 已知风险与应对

| 风险 | 应对 |
|------|------|
| `ctx.roundRect` 在旧版浏览器不存在 | 所有圆角矩形均用 `arcTo` 手动实现，已规避 |
| AudioContext 被浏览器静音（自动播放策略） | AudioManager 懒加载，首次用户交互后才初始化 |
| `Math.min(...大数组)` 栈溢出 | 使用 `reduce` 代替展开运算符 |
| 平台数组为空时 `reduce` 返回 Infinity | 已在 spawnPlatform 中处理（初始化保证数组非空）|
| 分数在 cameraY 为正值时计算出负数 | 使用 `Math.min(cameraY, 0)` 确保只在上升阶段计分 |
