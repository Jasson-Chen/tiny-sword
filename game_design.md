# Tiny Swords Roguelite - 游戏完整重建文档

> 本文档包含从零重建该游戏所需的全部信息。将本文档提供给 AI agent，配合 Tiny Swords (Free Pack) 素材包，即可生成完整可运行的游戏。

---

## 一、项目概述

- **游戏类型**：俯视角动作割草 Roguelite
- **引擎**：Phaser 3（v3.80+）
- **构建工具**：Vite 5
- **语言**：JavaScript（ES Modules）
- **素材包**：Tiny Swords (Free Pack)，像素风格，64×64 tile 基础单位
- **运行方式**：`npm run dev` 启动 Vite 开发服务器，浏览器访问

---

## 二、项目结构

```
tiny-swords-game/
├── index.html                    # 入口 HTML
├── package.json                  # 依赖：phaser ^3.80.0, vite ^5.0.0
├── public/
│   └── assets/ → 符号链接到素材包目录 "Tiny Swords (Free Pack)"
└── src/
    ├── main.js                   # Phaser 配置与启动
    ├── scenes/
    │   ├── BootScene.js          # 素材加载 + 动画创建
    │   ├── GameScene.js          # 主游戏场景（地图/玩家/敌人/碰撞/装饰）
    │   └── UIScene.js            # HUD + 升级面板 + 商店 + 死亡结算
    ├── entities/
    │   ├── Player.js             # 玩家角色
    │   └── Enemy.js              # 敌人基类
    ├── systems/
    │   └── RoomManager.js        # 房间状态管理 + 刷怪
    └── utils/
        └── MapGenerator.js       # 随机地图生成
```

---

## 三、Phaser 配置

```javascript
{
  type: Phaser.AUTO,
  width: window.innerWidth,
  height: window.innerHeight,
  backgroundColor: '#3b8686',
  physics: { default: 'arcade', arcade: { debug: false } },
  scene: [BootScene, GameScene, UIScene],
  scale: { mode: Phaser.Scale.RESIZE, autoCenter: Phaser.Scale.CENTER_BOTH },
  pixelArt: true
}
```

---

## 四、素材使用规则

### 4.1 素材路径前缀

所有素材位于：`Tiny Swords.zip` 请仔细解压并放置好素材的位置以便后续调用

### 4.2 Tilemap 结构

文件：`Terrain/Tileset/Tilemap_color1.png`（576×384），每 tile 64×64，共 9 列 × 6 行。
Phaser 中作为 spritesheet 加载，frame index = row × 9 + col。

#### 草地 Tile（左 Block，cols 0-2）

| | col 0 | col 1 | col 2 |
|---|---|---|---|
| row 0 | 左上角 | 上边缘 | 右上角 |
| row 1 | 左边缘 | **中心填充** | 右边缘 |
| row 2 | 左下角 | 下边缘 | 右下角 |

**关键规则**：col 3 的 tile 不可用于凹角。当四个方向邻居都有草地但对角线缺失时，直接使用中心填充 tile (col=1, row=1, frame=10)。

#### 悬崖墙壁 Tile（右 Block，row 4）

| col 5 (frame=41) | col 6 (frame=42) | col 7 (frame=43) |
|---|---|---|
| 悬崖墙-左端 | 悬崖墙-中间 | 悬崖墙-右端 |

#### 草地 Tile 选择逻辑

检查上(N)、下(S)、左(W)、右(E) 四个方向邻居：

```
N+W 缺失 → (0,0)    N+E 缺失 → (2,0)
S+W 缺失 → (0,2)    S+E 缺失 → (2,2)
仅 N 缺失 → (1,0)   仅 S 缺失 → (1,2)
仅 W 缺失 → (0,1)   仅 E 缺失 → (2,1)
全部存在 → (1,1)     // 无论对角线是否缺失
```

#### 悬崖墙壁选择逻辑

对南边缘格子（`grid[r][c]=1` 且 `grid[r+1][c]=0`），在 `(c, r+1)` 位置画悬崖：

```
左右都是南边缘 → frame 42（中间）
仅右侧是南边缘 → frame 41（左端）
仅左侧是南边缘 → frame 43（右端）
两侧都不是     → frame 42（中间）
```

### 4.3 渲染顺序（depth 值）

1. **depth 0**：水面背景（`Water Background color.png`，64×64，铺满全图）
2. **depth 1**：水波泡沫（`Shadow.png` 192×192 + `Water Foam.png` 第一帧 192×192）+ 水中石头
3. **depth 2**：悬崖墙壁
4. **depth 3**：草地表面
5. **depth 5**：建筑和装饰物（树/岩石）
6. **depth 8**：掉落物
7. **depth 10**：玩家和敌人、箭矢

### 4.4 水波泡沫放置条件（必须全部满足）

1. 当前格子是水面（grid=0）
2. 不是悬崖墙壁占据的位置
3. 至少一个上下左右邻居是草地
4. 以当前格子为中心的 3×3 范围内没有任何草地格子

### 4.5 水中石头

素材：`Rocks in the Water/Water Rocks_01.png`，spritesheet 64×64，取第一帧。
放置条件：水面格子，5×5 范围内无草地。随机散布约 40 个。

### 4.6 加载的素材清单

**Tilemap/地形**：
- `water`：Water Background color.png（image）
- `tilemap`：Tilemap_color1.png（spritesheet 64×64）
- `foam`：Water Foam.png（spritesheet 192×192）
- `shadow`：Shadow.png（image）

**玩家（Blue Warrior）**：
- `warrior_idle/run/attack1/attack2/guard`（spritesheet 192×192）

**敌人（Red/Yellow/Purple/Black × Pawn/Archer/Warrior）**：
- `{pawn|archer|warrior}_{color}_{idle|run}`（spritesheet 192×192）
- `archer_{color}_shoot`（spritesheet 192×192）
- `warrior_{color}_attack`（spritesheet 192×192）

**其他**：
- `arrow`：Arrow.png（image 64×64）
- `castle/tower/house1`：建筑（image）
- `explosion1`：Explosion_01.png（spritesheet 192×192）
- `dust1`：Dust_01.png（spritesheet 64×64）
- `gold/meat`：资源（image）
- `rock1/rock2`：岩石（image 64×64）
- `water_rock`：水中石头（spritesheet 64×64）
- `tree1`：大绿松树（spritesheet 192×256）
- `tree3`：小黄树（spritesheet 192×192）
- `bar_base/bar_fill`：UI 血条

### 4.7 动画定义

| 动画 key | 素材 key | 帧率 | 循环 |
|---|---|---|---|
| player_idle | warrior_idle | 8 | -1 |
| player_run | warrior_run | 10 | -1 |
| player_attack1 | warrior_attack1 | 12 | 0 |
| player_attack2 | warrior_attack2 | 12 | 0 |
| player_guard | warrior_guard | 8 | -1 |
| {unit}_{color}_idle | 对应素材 | 8 | -1 |
| {unit}_{color}_run | 对应素材 | 8 | -1 |
| archer_{color}_shoot | 对应素材 | 10 | 0 |
| warrior_{color}_attack | 对应素材 | 12 | 0 |
| explosion | explosion1 | 12 | 0 |
| dust | dust1 | 10 | 0 |

---

## 五、地图生成系统

### 5.1 参数

- 地图尺寸：150×80 tiles（9600×5120 像素）
- 区域数量：最多 12 个
- 区域大小：18-26 × 10-15 tiles（约 3/4 浏览器全屏）
- 区域间距：至少 4 tiles
- 路径宽度：3 tiles
- 使用 seeded RNG（线性同余法）

### 5.2 生成流程

1. 随机放置矩形区域，检查不重叠（含 4 tile 间距）
2. 计算区域中心点
3. Kruskal MST 连接所有区域
4. 补充边确保每个区域至少 2 个连接
5. 画 L 形路径（先水平后垂直，3 tile 宽）
6. 分配区域类型

### 5.3 区域类型分配

- **spawn**：第一个生成的区域
- **boss**：距离 spawn 最远的区域
- **shop**：距离 spawn 中等距离的区域
- **elite**：距离 spawn 第二远的区域
- **battle**：其余所有区域

---

## 六、玩家系统

### 6.1 操作

| 操作 | 按键 |
|---|---|
| 移动 | WASD / 方向键 |
| 攻击 | 鼠标左键（空格备用）|
| 格挡 | 鼠标右键 |

右键菜单已禁用（`input.mouse.disableContextMenu()`）。

### 6.2 初始属性

```javascript
{ maxHp: 100, hp: 100, attack: 10, attackSpeed: 1, moveSpeed: 200, attackRange: 80, hpRegen: 0 }
```

### 6.3 物理碰撞体

- sprite 尺寸 192×192，碰撞体 48×48，offset (72, 100)

### 6.4 攻击机制

- 交替播放 Attack1/Attack2 动画
- 攻击后 150ms 检测范围内敌人并造成伤害
- 冷却时间 = 1000 / attackSpeed (ms)
- 攻击方向基于 flipX 朝向，检测点在角色前方 40px

### 6.5 受击机制

- 格挡时减伤 70%
- 闪白 100ms（`setTintFill(0xffffff)`）
- 无敌帧 500ms
- HP 归零触发 playerDeath 事件

### 6.6 朝向

- 移动时根据水平方向设置 `flipX`：向左移动时 `flipX = true`（`vx < 0`）
- 敌人同理：`dx < 0` 时 flipX

---

## 七、敌人系统

### 7.1 基础属性

| 类型 | HP | 攻击 | 速度 | 范围 | 行为 |
|---|---|---|---|---|---|
| pawn | 20 | 5 | 80 | 40 | 近战 |
| archer | 15 | 8 | 60 | 250 | 远程 |
| warrior | 40 | 10 | 70 | 45 | 近战 |

### 7.2 颜色倍率

| 颜色 | HP 倍率 | 攻击倍率 |
|---|---|---|
| red | 1.0 | 1.0 |
| yellow | 1.5 | 1.3 |
| purple | 2.0 | 1.6 |
| black | 3.0 | 2.0 |

### 7.3 AI 行为

**近战（pawn/warrior）**：
- 距离 > range×0.8：向玩家移动，播放 run 动画
- 距离 ≤ range：停下，冷却结束后攻击（1500ms CD）
- warrior 攻击时播放 attack 动画

**远程（archer）**：
- 100 < 距离 < range：停下射箭（2000ms CD）
- 距离 ≤ 100 或 > range：向玩家移动

### 7.4 箭矢

- 速度 300 px/s，scale 1.5，碰撞体 32×32，depth 10
- 朝向玩家方向飞行，rotation = angle
- 命中玩家后销毁，3 秒后自动销毁

### 7.5 受击与死亡

- **受击**：闪白 80ms + Dust 粒子特效
- **死亡**：禁用物理体 → Explosion 特效 → alpha 渐隐 300ms → 触发 enemyKilled 事件 → 销毁 → 掉落资源

### 7.6 掉落

- 50% 概率掉金币（value=10）
- 15% 概率掉肉（value=20，回血）

---

## 八、房间管理系统

### 8.1 房间状态

`locked` → `active` → `cleared`

spawn 房间初始为 cleared。

### 8.2 进入房间逻辑

每帧检测玩家像素坐标是否在某个区域矩形内。进入新房间时：
- battle/elite/boss：500ms 后开始刷怪
- shop：直接标记 cleared，暂停游戏，弹出商店面板

### 8.3 刷怪机制

- 分 2-3 波，每波间隔 1500ms
- 敌人随机生成在区域内部（边缘各缩 1 tile）
- 全部清完后触发 roomCleared 事件

### 8.4 难度递增

敌人颜色根据已清房间数递增：
```javascript
const colors = ['red', 'red', 'yellow', 'yellow', 'purple', 'purple', 'black'];
const baseColor = colors[Math.min(cleared, colors.length - 1)];
const hardColor = colors[Math.min(cleared + 1, colors.length - 1)];
```

### 8.5 波次配置

**battle 房间**：
1. 5 pawn (baseColor)
2. 2 pawn + 2 archer (baseColor)
3. 1 warrior (hardColor) + 2 pawn (baseColor)

**elite 房间**：
1. 2 warrior + 2 archer (hardColor)
2. 3 warrior + 3 pawn (hardColor)

**boss 房间**（固定 black）：
1. 4 pawn + 2 archer
2. 4 warrior
3. 3 warrior + 3 archer + 2 pawn

---

## 九、碰撞系统

### 9.1 水面碰撞墙

遍历所有水面格子，仅对与草地相邻的水面格子创建不可见矩形碰撞体（优化性能）。
玩家和敌人都与 wallGroup 碰撞。

### 9.2 装饰物碰撞

树和岩石加入 wallGroup，碰撞体尺寸：
- 大绿树：body 64×48，offset (64, 180)
- 小黄树：body 64×48，offset (64, 120)
- 岩石：body 48×32，offset (8, 20)

---

## 十、装饰物放置规则

### 10.1 放置条件

1. 必须在内部草地 tile 上（四个方向邻居都是草地）
2. 不在出入口附近（区域边缘外 2 格范围内的草地 tile 及其周围 2 格都排除）
3. 不在区域中心 2 格范围内
4. 相邻装饰物间距至少 3 tiles
5. 总数约 80 个

### 10.2 类型轮换

```
count % 4 == 0 → 大绿树 (tree1)
count % 4 == 1 → 小黄树 (tree3)
count % 4 == 2 → 岩石1 (rock1)
count % 4 == 3 → 岩石2 (rock2)
```

---

## 十一、建筑放置

根据区域类型在区域上方放置建筑：
- spawn → castle
- shop → house1
- elite → tower

位置：区域水平中心，垂直 row+1 位置。

---

## 十二、拾取系统

- 掉落物 scale 0.4，depth 8
- 生成时有弹跳动画（y-20, 200ms, yoyo, Bounce ease）
- 玩家与 pickupGroup overlap 触发收集
- gold：增加 gameStats.gold，触发 goldChanged 事件
- meat：回复 HP（不超过 maxHp），触发 playerHit 事件更新 UI

---

## 十三、UI 系统

### 13.1 HUD（常驻，scrollFactor=0）

- 左上 (20,40)：HP 标签 22px + 血条 250×28（bg #333, fill 绿/黄/红根据百分比）+ 数值文字 18px
- 左上 (20,75)：金币 💰 22px 金色
- 左上 (20,110)：击杀 ⚔ 22px 白色

### 13.2 升级面板

- 触发：roomCleared 事件 → 暂停 Game 场景
- 半透明黑色遮罩 + "选择升级" 标题 28px 金色
- 从 6 个升级中随机选 3 个，横排显示为按钮（160×60，蓝色）
- 点击后应用升级、隐藏面板、恢复 Game 场景

### 13.3 升级选项

| 名称 | 效果 |
|---|---|
| 攻击力 +20% | attack × 1.2 |
| 攻速 +15% | attackSpeed × 1.15 |
| 移动速度 +10% | moveSpeed × 1.1 |
| 攻击范围 +25% | attackRange × 1.25 |
| 最大生命 +30% | maxHp × 1.3，回满血 |
| 生命恢复 | hpRegen + 1（每秒回复） |

### 13.4 商店面板

- 触发：进入 shop 区域 → 暂停 Game 场景
- 3 个商品 + 离开按钮
- 回满血 50 金币 / 攻击力 +5% 100 金币 / 最大生命 +10% 100 金币
- 金币不足时按钮灰色不可点击

### 13.5 死亡结算

- 触发：playerDeath 事件
- 黑色遮罩 0.7 + "你已阵亡" 36px 红色
- 显示击杀/金币/房间/时间统计
- "重新开始" 按钮 → 停止 UI 和 Game 场景，重新启动 Game

---

## 十四、事件系统

| 事件 | 触发时机 | 处理 |
|---|---|---|
| playerHit | 玩家受伤/回血 | UI 更新血条 |
| playerDeath | 玩家 HP=0 | UI 显示死亡结算 |
| enemyKilled | 敌人死亡 | RoomManager 计数 + gameStats.kills++ |
| roomCleared | 房间清完 | 暂停游戏 + UI 显示升级面板 |
| goldChanged | 金币变化 | UI 更新金币显示 |
| enterShop | 进入商店 | UI 显示商店面板 |

---

## 十五、游戏统计

```javascript
gameStats = { kills: 0, gold: 0, roomsCleared: 0, startTime: Date.now() }
```

---

## 十六、已知限制与注意事项

1. 素材包路径含空格 "Tiny Swords (Free Pack)"，需要通过符号链接或正确编码处理
2. 所有角色只有左右两个朝向（通过 flipX 实现），flipX 逻辑：向左移动/面向左侧时 `flipX = true`（即 `vx < 0` 或 `dx < 0`）
3. 凹角位置（四方向邻居都有草地但对角线缺失）直接使用中心 tile，不使用 tilemap col 3/col 8 的 tile
4. 悬崖墙壁只用一行（row 4），不用 row 5
5. 水波泡沫的 3×3 范围不能覆盖任何草地格子
6. 装饰物必须远离出入口（区域边缘外草地 tile 周围 2 格范围排除）
7. 升级选择和商店期间游戏暂停（`scene.pause('Game')` / `scene.resume('Game')`）
