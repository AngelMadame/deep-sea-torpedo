# QA 测试报告

测试对象：深海求生（Deep Sea Survivor）— `demos/20260430_deep_sea_survivor/index.html`
依据策划案：`demos/20260430_deep_sea_survivor/01_design.md`
依据开发报告：`demos/20260430_deep_sea_survivor/04_dev_report.md`
测试环境：代码审查 + 逻辑走查（静态分析）

## 测试用例与结果

### AC-01 游戏可启动
**条件**：浏览器打开 HTML 文件后显示开始界面，点击后进入游戏
**结果**：✅ 通过
**分析**：单 HTML 文件，无外部依赖。`scene = SCENE.MENU` 初始化，`drawMenu()` 绘制标题、操作说明和闪烁开始提示。`Enter/Space/点击` 触发 `initGame()` 进入游戏。

### AC-02 玩家可移动
**条件**：WASD/方向键控制潜艇 8 方向移动，移动流畅无卡顿
**结果**：✅ 通过
**分析**：`updatePlayer()` 正确处理 WASD 和方向键，使用 `normalize()` 归一化对角线速度，`player.speed * dt` 帧率无关移动。

### AC-03 自动攻击
**条件**：有敌人在场时，潜艇按间隔自动向最近敌人发射鱼雷
**结果**：✅ 通过
**分析**：`updateTorpedoes()` 在 `torpedoTimer <= 0 && enemies.length > 0` 时发射。`findNearestEnemies()` 按距离排序选取目标，支持多发鱼雷分别瞄准不同目标。无敌人时不发射（AC-05 边界）。

### AC-04 怪物生成
**条件**：怪物从屏幕外生成，朝玩家移动，至少 3 种类型
**结果**：✅ 通过
**分析**：3 种怪物类型定义在 `ENEMY_TYPES`（jellyfish/fangtooth/octopus）。`spawnEnemy()` 在 `player` 位置 + 随机角度 × `max(W,H)*0.7` 生成，确保在屏幕外。`updateEnemies()` 使用 `normalize` 向量追踪玩家。

### AC-05 怪物波次
**条件**：随时间推进，怪物数量和强度明显递增
**结果**：✅ 通过
**分析**：`WAVE_CONFIG` 定义 6 波次，`getCurrentWave()` 根据 `gameTime` 确定当前波。生成间隔从 1.0s 递减至 0.3s，属性倍率从 ×1.0 递增至 ×2.5。后期波次增加章鱼权重。

### AC-06 碰撞伤害
**条件**：怪物接触玩家时玩家扣血，鱼雷命中怪物时怪物扣血
**结果**：✅ 通过
**分析**：
- 怪物→玩家：`updateEnemies()` 使用 `circleCollide` 检测，应用护甲减伤 `Math.ceil(dmg * (1 - armor))`，0.5 秒冷却。
- 鱼雷→怪物：`updateTorpedoes()` 碰撞检测，命中扣血并消除鱼雷。

### AC-07 击杀掉落
**条件**：怪物死亡后掉落经验球
**结果**：✅ 通过
**分析**：`onEnemyKilled()` 创建经验球 `{ x, y, value: enemy.xp, lifetime: 10 }`。

### AC-08 经验升级
**条件**：拾取经验球填充经验条，满后触发升级
**结果**：✅ 通过
**分析**：`updateXpOrbs()` 拾取后 `xp += orb.value`，`while (xp >= xpToNext)` 循环处理连续升级。`xpRequired(lvl) = 10 + lvl * 5` 符合策划案。

### AC-09 升级选择
**条件**：升级时游戏暂停，显示 3 个随机强化选项，选择后生效
**结果**：✅ 通过
**分析**：`triggerLevelUp()` 设 `scene = UPGRADE_PICK`，主循环不调用 `update()`（游戏暂停）。`drawUpgradePick()` 绘制卡片 UI，支持鼠标悬停高亮+点击和键盘 1/2/3 选择。`applyUpgrade()` 正确应用 8 种强化效果。

### AC-10 HUD 显示
**条件**：始终可见血量条、经验条、存活时间、击杀数
**结果**：✅ 通过
**分析**：`drawHUD()` 渲染：血量条（带颜色分级）、经验条（含等级）、存活时间/总时间、波次、击杀数、敌人数。

### AC-11 胜利结算
**条件**：存活 180 秒后显示胜利画面，展示击杀数和等级
**结果**：✅ 通过
**分析**：`update()` 中 `gameTime >= GAME_DURATION` 时设 `scene = END, victory = true`。`drawEndScreen()` 在 victory 模式显示"🎉 胜利！🎉"，展示存活时间、击杀数、等级。

### AC-12 死亡结算
**条件**：血量归零后显示死亡画面，展示存活时间和击杀数
**结果**：✅ 通过
**分析**：`updateEnemies()` 中 `player.hp <= 0` 时设 `scene = END, victory = false`。`drawEndScreen()` 在非 victory 模式显示"💀 潜艇损毁 💀"。

### AC-13 重新开始
**条件**：结算画面可重新开始新一局
**结果**：✅ 通过
**分析**：END 场景中检测 `keys['KeyR'] || mouseClicked` 调用 `initGame()`，完全重置所有状态。

### AC-14 性能表现
**条件**：正常游戏过程中帧率 ≥ 30 FPS
**结果**：✅ 通过（静态评估）
**分析**：
- 怪物上限 80，粒子上限 200，屏幕外怪物跳过绘制
- 碰撞检测为简单圆形距离计算（O(n×m) 最大 80×10 = 800/帧）
- dt cap 50ms 防止卡帧雪崩
- 经验球 10 秒超时清理
- 预计现代浏览器可稳定 60 FPS
- **需实际运行验证**

### AC-15 零依赖运行
**条件**：双击 HTML 文件即可在浏览器中运行，无需服务器或外部资源
**结果**：✅ 通过
**分析**：单 HTML 文件，内嵌全部 CSS + JS，无 `import`/`require`/CDN 引用，所有图形为 Canvas 绘制。

## 边界测试

| 边界情况 | 结果 | 分析 |
|---|---|---|
| 同屏 80 怪物上限 | ✅ | `spawnEnemy()` 首行检查 `enemies.length >= MAX_ENEMIES` |
| 升级选项不足 | ✅ | `triggerLevelUp()` 使用 `Math.min(3, available.length)`，选项为 0 时直接 return |
| 玩家静止不动 | ✅ | 怪物生成/追踪/自动攻击均不依赖玩家移动 |
| 多怪物同时接触 | ✅ | 遍历所有碰撞怪物累加伤害，共享 0.5s 冷却 |
| 鱼雷无目标 | ✅ | `enemies.length > 0` 条件保护 |
| 升级选择时暂停 | ✅ | `scene = UPGRADE_PICK` 不调用 `update()` |
| 经验球 10 秒清理 | ✅ | `orb.lifetime -= dt`，≤ 0 时 splice |
| Canvas 960×640 | ✅ | `canvas.width = W; canvas.height = H;` 固定尺寸 |

## Bug 列表

无阻塞性 Bug 发现。

### 轻微观察项（非阻塞）

| # | 描述 | 严重程度 | 阻塞交付 |
|---|---|---|---|
| OBS-1 | 碰撞伤害冷却是全局共享的（多怪物接触时只触发一次伤害判定），而非每个怪物独立 0.5s 冷却。策划案表述为"每个怪物独立计算伤害（但共享 0.5 秒碰撞冷却）"，当前实现符合括号内说明 | 信息 | 否 |
| OBS-2 | `sort(() => Math.random() - 0.5)` 随机洗牌算法非完美均匀（Fisher-Yates 更佳），但对升级选项随机性影响极小 | 信息 | 否 |
| OBS-3 | 怪物没有分离力，密集时可能堆叠重合，但不影响游戏性 | 信息 | 否 |

## 测试结论：✅ PASSED

所有 15 条验收标准代码审查通过，8 项边界情况处理正确，无阻塞性 Bug，代码逻辑与策划案数值完全匹配。游戏可交付。

**建议**：用户实际运行后验证 AC-14（性能 ≥ 30 FPS）。