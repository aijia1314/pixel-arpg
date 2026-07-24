# 像素近战 ARPG — 手机端网页游戏

> 横屏 2D 像素风近战动作游戏，单文件 HTML5（Canvas），适配手机触控。

## 快速启动

```bash
# 在项目目录下启动任意 HTTP 服务器
cd arpg
python -m http.server 8080
# 手机同 Wi-Fi 下访问 http://<电脑IP>:8080
# 电脑直接打开 http://localhost:8080
```

**单文件**：`index.html` 包含全部 HTML / CSS / JavaScript，可直接粘贴到 CodePen 运行。

---

## 架构总览

```
┌─────────────────────────────────────────────────┐
│  index.html (单文件)                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ CSS 层    │  │ UI 层    │  │ 游戏逻辑层     │  │
│  │ 全屏黑底  │  │ Canvas   │  │ 更新 + 渲染   │  │
│  │ 禁止滚动  │  │ 摇杆/按钮│  │ 敌人AI/战斗   │  │
│  │ 横屏检测  │  │ HUD/暗角 │  │ 粒子/震屏     │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────┘
```

### 双 Canvas 渲染管线

```
游戏逻辑更新 → 离屏Canvas(480×270) → scale到屏幕Canvas → UI叠加
                  ↑ 像素风低分辨率        ↑ 含震动偏移        ↑ 摇杆/按钮/HUD
```

- **`offscreen` (g)**：480×270 内部分辨率，所有游戏实体在此绘制
- **`canvas` (ctx)**：屏幕分辨率，先 `scale` 绘制离屏画布，再叠加 UI 层
- **缩放模式**：`Math.max(sw/GW, sh/GH)` cover 填满，横屏无黑边

### 游戏循环

```
requestAnimationFrame
  → 清理死亡敌人
  → update(dt)       // 逻辑：移动/战斗/AI/粒子
  → drawGameWorld()  // 离屏Canvas：背景/角色/特效
  → drawUI()         // 屏幕Canvas：UI叠加
```

---

## 坐标系

| 坐标系 | 原点 | 单位 | 用途 |
|--------|------|------|------|
| 游戏世界 | 左上(0,0), 480×270 | 像素 | 实体位置、碰撞、移动 |
| 屏幕 | 左上(0,0), sw×sh | 像素 | 摇杆、按钮、HUD |
| 角度 | 正右=0, 顺时针 | 弧度 | `player.facing`, 攻击方向 |

---

## 玩家系统

### 状态对象 (player)

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `x, y` | float | GW/2, GH/2 | 世界坐标 |
| `w, h` | int | 18, 22 | 碰撞体积 |
| `speed` | float | 100 | 最大速度 px/s |
| `vx, vy` | float | 0 | 当前平滑速度 |
| `facing` | rad | 0 | 朝向角（右=0） |
| `walkFrame` | float | 0 | 走路动画计时器 |
| `attackCD` | sec | 0 | 普攻冷却剩余 |
| `attackTimer` | sec | 0 | 刀光视觉持续 |
| `skillCD` | sec | 0 | 技能冷却剩余 |
| `isDashing` | bool | false | 冲刺中 |
| `dashTimer` | sec | 0 | 冲刺剩余时间 |
| `dashDx, dashDy` | float | 0 | 冲刺方向 |
| `hp` | int | 100 | 生命值 |
| `invincible` | sec | 0 | 受伤无敌 |
| `hitstop` | sec | 0 | 打击停顿冻结 |
| `hurtFlash` | sec | 0 | 受伤闪白 |

### 移动系统（Lerp 平滑）

```
摇杆输入 → 死区过滤(3%) → 二次响应曲线 → 目标速度
→ player.vx += (target - vx) * (1 - e^(-40·dt))  // 指数平滑
→ player.x += vx * dt
```

关键参数：`deadZone=0.03`, `lerpSpeed=40`, 起步瞬间给 30% 初速消除迟滞。

### 自动索敌

玩家不移动时，若 75px 内有敌人，自动面向最近敌人。

---

## 敌人系统

### 状态对象 (enemy)

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `x, y` | float | 随机边缘 | 世界坐标 |
| `w, h` | int | 16, 16 | 碰撞体积 |
| `hp` | int | 50 | 生命值 |
| `maxHp` | int | 50 | 最大生命 |
| `speed` | float | 40-65 | 追踪速度 |
| `stun` | sec | 0 | 硬直计时 |
| `hitstop` | sec | 0 | 打击停顿 |
| `flashTimer` | sec | 0 | 受击闪白 |
| `knockbackVx/Vy` | float | 0 | 击退速度 |
| `bounceT` | float | 随机 | 弹跳动画相位 |
| `colorVar` | int | 0-2 | 颜色变体 |

### AI 行为

1. 追向玩家 `angleTo(enemy, player)`
2. 硬直/打击停顿期间停止 AI 移动（仅应用击退）
3. 接触玩家造成 8 伤害，推开玩家，玩家 0.4s 无敌
4. 死亡 → 粒子爆炸 → 2s 后在边缘重生
5. 始终维持 `MAX_ENEMIES=3` 个敌人

---

## 战斗系统

### 普攻（Arc Slash）

| 参数 | 值 | 说明 |
|------|-----|------|
| 冷却 | 0.3s | `player.attackCD` |
| 刀光持续 | 0.15s | `player.attackTimer` |
| 扇形角度 | 120° (2π/3) | `arcAngle` |
| 范围 | 38px | 检测半径 |
| 伤害 | 25 | 每次命中 |

触发：按下普攻键 → `performAttack()` → 遍历敌人 → 扇形判定 → `applyDamage()`

### 技能（Dash Slash / 冲刺斩）

| 参数 | 值 | 说明 |
|------|-----|------|
| 冷却 | 3.0s | UI 显示倒计时遮罩 |
| 冲刺时长 | 0.18s | 2.8× 速度 |
| 伤害 | 35 | 路径上敌人 |
| 范围 | 38px | 碰撞检测 |

### 伤害处理 applyDamage()

每次命中触发连锁反馈：

```
applyDamage(enemy, damage, attackAngle)
  ├─ 扣血 + 闪红 0.1s (enemy.flashTimer)
  ├─ 硬直 0.2s   (enemy.stun)
  ├─ 击退 180px/s (enemy.knockbackVx/Vy)
  ├─ 打击停顿 0.08s (player.hitstop + enemy.hitstop)
  ├─ 屏幕震动 0.1s  (screenShake ±5px)
  ├─ 黄色火花 10粒 + 白色火花 5粒
  └─ 死亡 → 爆炸粒子 30粒 + 重生队列
```

---

## 打击感系统（六层反馈）

| 层级 | 机制 | 参数 | 函数位置 |
|------|------|------|---------|
| 1. 打击停顿 | 玩家+目标冻结 | 0.08s | `player.hitstop`, `enemy.hitstop` |
| 2. 受击闪白 | 红/白交替 | 0.1s | `enemy.flashTimer` |
| 3. 硬直 | 停止AI移动 | 0.2s | `enemy.stun` |
| 4. 击退 | 沿攻击方向弹开 | 180px/s | `enemy.knockbackVx/Vy` |
| 5. 屏幕震动 | 随机±5px偏移 | 0.1s | `screenShake` |
| 6. 打击火花 | 粒子飞溅 | 0.22s寿命 | `spawnSparks()` |

---

## 输入系统

### 浮动摇杆（左半屏 55%）

```
touchstart → 设定底座位置
touchmove → 计算偏移 → 超出半径 → 底座跟随手指移动
          → 输出归一化方向 joystick.dx/dy ∈ [-1, 1]
touchend  → 复位到底部默认位置
```

### 按钮（右半屏）

- 普攻键：右下角大圆，半径 `BTN_RADIUS_L`（屏幕短边 10.5%）
- 技能键：右下角偏左小圆，半径 `BTN_RADIUS_S`（屏幕短边 7.8%）
- 触控区扩大 1.3 倍（`hitButton` 函数）
- 技能冷却时显示扇形遮罩 + 秒数

### 鼠标支持

桌面端鼠标操作同步映射（`mousedown/mousemove/mouseup`）。

---

## UI 系统

### 屏幕布局（横屏）

```
┌─────────────────────────────────────────────┐
│ HP 100/100                                  │
│                                         敌人:3│
│                                             │
│          [ 游戏画面 480×270 scaled ]         │
│                                             │
│  ◉ 摇杆                        ⚔普攻       │
│  (左)                          💨技(CD)      │
└─────────────────────────────────────────────┘
```

### 动态 UI 尺寸 (recalcUI)

所有 UI 尺寸基于屏幕短边的百分比，每次 resize 重算：

| 元素 | 比例 | 公式 |
|------|------|------|
| 摇杆半径 | 12% | `Math.round(ref * 0.12)` |
| 普攻键半径 | 10.5% | `Math.round(ref * 0.105)` |
| 技能键半径 | 7.8% | `Math.round(ref * 0.078)` |

### 竖屏检测

`isLandscape = sw > sh`，竖屏时：
- 显示 📱 "请旋转手机至横屏" 提示
- 隐藏 canvas，暂停渲染（游戏逻辑继续跑）

---

## 粒子系统

### 数据结构

```js
{ x, y, vx, vy, life, maxLife, color, size }
```

- 速度：60-260 px/s 随机方向
- 寿命：0.1-0.22s
- 颜色：hex 字符串（`#ffdd44`, `#ffffff`, `#ff6644` 等）
- 渲染：带 alpha 衰减的纯色矩形

### 生成场景

| 事件 | 数量 | 颜色 |
|------|------|------|
| 普攻命中 | 10+5 | 黄 + 白 |
| 技能命中 | 10+5 | 黄 + 白 |
| 敌人死亡 | 20+10 | 橙红 + 橙 |
| 玩家死亡 | 30+20+10 | 红 + 橙 + 白 |

---

## 角色绘制

### 骑士 drawKnight()

4 方向（右/下/左/上），走路摆腿动画，带：
- 银灰铠甲、头盔面罩、剑、盾
- 受伤闪白（`hurtFlash` 控制闪烁频率）
- 冲刺淡蓝染色（`isDashing`）

### 史莱姆 drawSlime()

弹跳压扁动画（`bounceT` 驱动 `sin` 变形），3 色变体（绿/蓝/橙），带：
- 身体高光、亮斑、眼睛、小嘴
- 受伤表情变化（低血量）
- 受击闪白

---

## 修改指南

### 调整难度

```js
// 敌人属性 — spawnEnemy()
hp: 50,           // 生命值
speed: 40 + Math.random() * 25,  // 速度范围

// 玩家属性
player.speed: 100,  // 移动速度
player.hp: 100,     // 初始生命

// 伤害值 — applyDamage() / performAttack()
applyDamage(en, 25, player.facing);  // 普攻伤害
applyDamage(en, 35, player.facing);  // 技能伤害
player.hp -= 8;  // 敌人碰撞伤害

// 敌人数量
MAX_ENEMIES = 3;

// 冷却时间
player.attackCD = 0.3;  // 普攻冷却
player.skillCD = 3.0;   // 技能冷却
```

### 调整打击感

```js
// applyDamage() 中：
enemy.hitstop = 0.08;   // 停顿秒数（越大越"卡肉"）
screenShake = 0.1;      // 震动秒数
enemy.stun = 0.2;       // 硬直秒数
knockbackForce = 180;   // 击退力度 px/s
```

### 调整摇杆手感

```js
var deadZone = 0.03;    // 死区（越小越灵敏）
var t = 1 - Math.exp(-40 * dt);  // Lerp 速度（越大越灵敏）
// 起步初速：
player.vx = targetVx * 0.3;  // 0.3=30%初速
```

### 添加新敌人类型

1. 在 `spawnEnemy()` 中添加新模板（可基于 `Math.random()` 选择类型）
2. 在 `drawSlime()` 旁添加新绘制函数
3. 在敌人渲染循环中根据类型调用不同绘制函数

### 添加玩家新技能

1. 在 `player` 对象添加冷却/状态属性
2. 在 UI 层添加新按钮 + 输入处理
3. 在 `update()` 中添加技能逻辑
4. 在 `drawGameWorld()` 中添加视觉效果

---

## 浏览器兼容

- 使用 `var`（非 ES6 `let/const`）确保旧手机浏览器兼容
- 字符串拼接（非模板字面量）避免解析问题
- `requestAnimationFrame` + `performance.now()` 计时
- `Math.hypot()` 可能在极旧浏览器不可用（iOS 8+ / Android 5+ 均支持）

## 文件结构

```
arpg/
├── index.html    # 完整游戏（HTML+CSS+JS 单文件）
└── CLAUDE.md     # 本文档
```
