# 架构评估报告

评估对象：深海求生（Deep Sea Survivor）— 类吸血鬼幸存者 Web 游戏 Demo
关联策划案：`demos/20260430_deep_sea_survivor/01_design.md`（已通过 Gate 1，用户已确认）

## 功能可行性

**结论：完全可行**

策划案要求的所有功能均可通过纯 JavaScript + Canvas 2D API 实现：

| 功能 | 可行性 | 说明 |
|---|---|---|
| 游戏主循环 (60FPS) | ✅ | `requestAnimationFrame` 标准方案 |
| 2D 渲染（几何图形） | ✅ | Canvas 2D `fillRect`/`arc`/`stroke` 即可 |
| 键盘输入（WASD/方向键） | ✅ | `keydown`/`keyup` 事件标准方案 |
| 碰撞检测（圆形） | ✅ | 简单距离计算，无需物理引擎 |
| 怪物寻路（朝玩家移动） | ✅ | 向量归一化追踪，无需 A* |
| 摄像机跟随 | ✅ | Canvas `translate` 偏移 |
| 粒子效果 | ✅ | 简单粒子池，Canvas 绘制 |
| 暂停/恢复 | ✅ | 游戏状态机控制 update 停止 |
| HUD 覆盖渲染 | ✅ | Canvas 固定坐标绘制（不受摄像机偏移影响） |

**零外部依赖完全满足**——所有功能均由浏览器原生 API 提供。

## 当前代码影响

这是全新 Demo，不存在现有代码。无需考虑向后兼容或回归风险。

## 涉及模块

建议采用**单 HTML 文件内嵌 JavaScript** 架构，逻辑上按功能分区组织代码：

```
index.html（单文件）
├── <canvas> 元素
└── <script>
    ├── [CONFIG]        — 全局常量与数值配置
    ├── [UTILS]         — 数学工具（向量、距离、随机、碰撞）
    ├── [INPUT]         — 键盘输入管理
    ├── [CAMERA]        — 摄像机跟随
    ├── [PLAYER]        — 玩家潜艇（状态、移动、受伤）
    ├── [PROJECTILE]    — 鱼雷 & 声波脉冲
    ├── [ENEMY]         — 怪物类型定义 & 行为
    ├── [WAVE]          — 波次管理 & 生成调度
    ├── [XP_ORB]        — 经验球
    ├── [UPGRADE]       — 升级系统 & 强化选项池
    ├── [PARTICLE]      — 粒子效果
    ├── [HUD]           — 血量条/经验条/计时器/击杀数
    ├── [SCENE]         — 场景管理（菜单/游戏中/升级选择/结算）
    ├── [RENDER]        — 统一渲染入口（背景/实体/HUD）
    └── [MAIN]          — 游戏主循环（init/update/draw）
```

## 新增模块

全部为新增（从零开始）：

1. **CONFIG** — 集中管理所有数值参数，方便调试
2. **UTILS** — `vec2` 操作、距离计算、随机数、碰撞检测
3. **INPUT** — 键盘状态追踪（按下/释放）
4. **CAMERA** — 世界坐标 ↔ 屏幕坐标转换
5. **PLAYER** — 玩家实体，含属性、移动逻辑、受伤/无敌帧
6. **PROJECTILE** — 鱼雷生命周期管理，声波脉冲逻辑
7. **ENEMY** — 3 种怪物的模板配置与运行时行为
8. **WAVE** — 波次状态机，控制生成时机和怪物组合
9. **XP_ORB** — 经验球生成、磁力吸附、拾取判定
10. **UPGRADE** — 强化选项池、随机抽选、效果应用、叠加上限
11. **PARTICLE** — 轻量粒子池（击杀爆散、受伤闪烁）
12. **HUD** — 4 项 UI 数据的实时渲染
13. **SCENE** — 4 种游戏状态（MENU → PLAYING → UPGRADE_PICK → END）
14. **RENDER** — 渲染层序管理，深海背景 + 粒子 + 实体 + HUD
15. **MAIN** — `requestAnimationFrame` 主循环与 delta time 计算

## 需要修改的模块

无（全新项目）。

## 数据结构影响

核心运行时对象设计：

```javascript
// 玩家
player = {
  x, y, hp, maxHp, speed, pickupRadius, collisionRadius,
  angle, invincibleTimer,
  torpedoDamage, torpedoInterval, torpedoCount, torpedoSpeed,
  armor, hasSonarPulse,
  sonarDamage, sonarRadius, sonarInterval, sonarTimer
}

// 怪物
enemy = {
  x, y, hp, maxHp, speed, damage, xp, type, radius,
  flashTimer
}

// 鱼雷
projectile = { x, y, vx, vy, damage, lifetime }

// 经验球
xpOrb = { x, y, value, lifetime, beingPulled }

// 粒子
particle = { x, y, vx, vy, life, maxLife, color, radius }

// 游戏状态
gameState = {
  scene, // MENU | PLAYING | UPGRADE_PICK | END
  time, kills, level, xp, xpToNext,
  wave, paused, victory
}
```

所有对象使用 **plain object + array pool** 模式，避免 GC 压力。

## 依赖风险

**无外部依赖** — 风险极低。

唯一依赖为浏览器原生 API：
- `Canvas 2D Context` — 所有现代浏览器均支持
- `requestAnimationFrame` — IE10+ / 所有现代浏览器
- `KeyboardEvent` — 通用支持

**风险评级：🟢 低**

## 性能风险

| 场景 | 风险 | 缓解措施 |
|---|---|---|
| 同屏 80 只怪物 | 中 | 策划案已设上限 80；使用简单距离碰撞（O(n) 鱼雷 × O(m) 怪物），80×10 = 800 次距离计算/帧，完全可接受 |
| 粒子效果 | 低 | 粒子池上限 200，复用对象避免 GC |
| 经验球累积 | 低 | 策划案已设 10 秒超时清理 |
| 背景粒子 | 低 | 仅需 20-30 个缓慢移动点 |
| Canvas 绘制调用数 | 中 | 预计每帧 ~200 次绘制调用（80 怪物 + 10 鱼雷 + 50 经验球 + 30 背景粒子 + 50 效果粒子 + HUD），Canvas 2D 完全可以承受 |

**总体性能评估：满足 30 FPS 目标有充足余量（预计可达 60 FPS）**

优化建议（非必须，按需启用）：
1. 怪物超出屏幕 1.5 倍范围时跳过绘制
2. 碰撞检测先做 AABB 粗筛
3. 粒子使用对象池避免频繁创建

## 兼容性风险

| 环境 | 兼容性 |
|---|---|
| Chrome 90+ | ✅ 完全支持 |
| Firefox 90+ | ✅ 完全支持 |
| Safari 14+ | ✅ 完全支持 |
| Edge 90+ | ✅ 完全支持 |
| IE | ❌ 不支持（不在目标范围内） |
| 移动端浏览器 | ⚠️ 可运行但无触控支持（明确不做） |

**风险评级：🟢 低**（目标为桌面 Web 浏览器，完全覆盖）

## 测试建议

1. **功能测试**：逐一验证 15 条验收标准（AC-01 ~ AC-15）
2. **边界测试**：
   - 同屏 80 怪物时的性能和生成停止
   - 升级选项耗尽时的 UI 显示
   - 连续快速升级（快速击杀大量怪物）
   - 玩家在角落被大量怪物包围
3. **性能测试**：
   - 在 Wave 6（最高密度）时检查帧率 ≥ 30 FPS
   - 长时间运行（完整 180 秒）无内存泄漏
4. **输入测试**：
   - 同时按下多个方向键
   - 升级选择时键盘和鼠标两种方式
5. **流程测试**：
   - 完整通关（胜利）→ 重新开始 → 再通关
   - 立即死亡 → 重新开始
   - 多次升级选择

## 评估结论：✅ APPROVED

技术方案完全可行，纯 JavaScript + Canvas 2D 能够实现策划案所有功能。项目规模适中（预计 1500~2500 行代码），性能风险可控，零外部依赖保证了运行环境的简洁性。

## 准入条件

无额外准入条件。可直接进入开发。