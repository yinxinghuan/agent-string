# Agent String — 项目上下文

## 基本信息

- Repo: `yinxinghuan/agent-string`，本地: `/Users/yin/code/games/agent-string/`
- 技术栈: **单文件纯 HTML** (`index.html`)，Canvas 2D，无框架，无构建步骤
- 同步规则: 每次改完必须 `cp index.html game.html`，然后 git add/commit/push
- Game ID (排行榜): `agent-string`
- 默认 UI 字体: Space Mono（FONTS 数组 index 2）

---

## 游戏机制

### 词的类型（gold 数组 type 字段）

| type | 颜色 | 效果 |
|------|------|------|
| 无/`'gold'` | 各组色 | +10分，计入 lap，参与 combo，核心金词 |
| `'time'` | teal `[0,170,150]` | +addTime秒，+5分，每圈可重复拾取 |
| `'penalty'` | 深玫红 `[175,35,80]` | -10分，-6秒，屏幕抖动，每圈可重复 |

### Lap 机制

收集完所有 8 个核心金词 → `checkLap()` → +20秒 + 递增奖励分（50/60/70…）→ 核心金词重置（`hy += totalPassageH` 推到下一圈）

### 词回收逻辑（update 循环）

- `approxSy < -80`：核心金词未拾取 → miss 标记
- `approxSy < -(lvLSPACE()*2)`：
  - 核心金词未找到 → 重置推下圈
  - 时间/惩罚词 → 无论如何重置推下圈（可再拾取）
  - 触发过的陷阱词 → 重置推下圈
  - 普通词 → 推下圈

### 通关判定

- **仅时间归零**触发结算（`timeLeft <= 0`）
- `showEnd()` 中判断显示「下一关」：`lapCount > 0 || coreFound/total >= 0.5`
- 否则只显示「再来一次」

### Surge 机制

- L1-L3：特定金词（`surge:true`）触发，`surgeState='active'`
- 生成关（L4+）：`surgeInterval` 定时自动触发
- **关键**：`surgeTimer -= sec` 必须在 `if(cfg.surgeInterval)` 块**外**，否则 L1-L3 冲刺永不结束

---

## 关卡配置

| 关卡 | layoutMode | time | scrollNorm | surgeSpeed | surgeDuration | geomCount |
|------|-----------|------|-----------|-----------|--------------|-----------|
| L1 INIT | prose | 90s | 62 | 520 | 0.7s | 1 |
| L2 SCAN | stream | 80s | 90 | 680 | 0.65s | 2 |
| L3 DEEP | verse | 70s | 44 | 420 | 0.75s | 3 |

### 功能性金词

- **L1**: `river`(+15s), `sequences`(+12s), `finite`(-10分-6s)
- **L2**: `momentum`(+14s), `robust`(+11s), `predicted`(-10分-6s)
- **L3**: `potential`(+16s), `sufficient`(+12s), `alone`(-10分-6s)

### Surge 触发词（`surge:true`）

- L1: `signal`, `patterns`
- L2: `gradient`, `output`
- L3: `emergence`, `complexity`

---

## 字体系统

- `FONT()` → 始终返回 `HEALTH_FONTS[4]`（IBM Plex Mono），用于**布局计算**，保持词位置稳定
- `renderFont()` → 根据 `signalHealth`(0–4) 返回不同字体，用于**canvas 渲染**
- `trapFontTimer > 0` 时 renderFont 返回 VT323（短暂闪烁）
- `HEALTH_FONTS`: `[VT323, Martian Mono, Space Mono, Roboto Mono, IBM Plex Mono]`
- **不能混用**：layout 用 `FONT()`，draw 用 `renderFont()`，否则词位置跳动

---

## 得分

| 事件 | 分值 |
|------|------|
| 核心金词 | +10 |
| 连击 ×3/5/7 | +15/25/35 |
| Combo | +30 |
| Lap 第N圈 | +(50 + N×10) |
| 时间词 | +5 |
| 惩罚词 | -10 |

---

## 排行榜

- API base: `https://games-api.xinghuan-yin.workers.dev`
- 提交: `POST /score` → `{game_id, telegram_id, name, avatar_url, score}`
- 全球榜: `GET /leaderboard?game_id=agent-string&limit=50`
- 好友榜: postMessage 拉联系人列表 → `GET /leaderboard?...&telegram_ids=...`
- Aigram 检测: URL query `telegram_id` + `api_origin`
- 游戏结束时自动调用 `submitScore(score)`

---

## 已知踩坑

1. **Surge 永不结束**：`surgeTimer -= sec` 必须在 `if(cfg.surgeInterval)` 块外
2. **字体混用**：`FONT()` 用于布局，`renderFont()` 用于渲染，不能互换
3. **combo 误触发**：功能词（time/penalty）在 `collectWord` 里 early return，不参与 combo 检查；combo 检查时也需过滤非 gold 类型
4. **文字选中**：所有 overlay 必须加 `user-select:none;-webkit-user-select:none`
5. **时间无限续命**：时间词不设上限是**故意的**，鼓励玩家在一关内刷高分
