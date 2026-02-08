# Crash 游戏 API

多人实时倍率游戏：倍率从 1.00x 开始上升，玩家在 Crash 前兑现获得当前倍率的赔付。支持最多 3 个投注槽位。

## 房间机制

系统提供 4 个独立房间，每个房间有不同的 RTP 配置：

| 房间 ID | RTP | 倍率范围 |
|---------|-----|----------|
| 0 | 99% | 1.1x - 10000x |
| 1 | 98% | 1.1x - 10000x |
| 2 | 97% | 1.1x - 10000x |
| 3 | 96% | 1.1x - 10000x |

玩家进入的房间由 Operator 配置决定，同一 Operator 下的所有玩家使用同一房间。

## 获取游戏配置

连接成功时 `ctx.data.config` 已包含游戏配置，也可使用 `common.getConfig` 接口获取：

```javascript
const result = await centrifuge.rpc('common.getConfig', {
    gameId: 'inhousegame:crash'
});
```

**响应**：

```json
{
    "config": {
        "gameId": "inhousegame:crash",
        "status": "active",
        "rtp": 97,
        "betInfo": [
            {
                "currency": "USDT",
                "currencyType": "crypto",
                "defaultBet": 1,
                "minBet": 0.1,
                "maxBet": 10000,
                "maxProfit": 100000
            }
        ],
        "gameParameters": {
            "roomId": 2
        },
        "maxSlots": 3,
        "bettingDuration": 5,
        "waitingDuration": 3,
        "minMultiplier": 1.1,
        "maxMultiplier": 10000
    }
}
```

**gameParameters 字段说明**（可由 Operator 配置）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `roomId` | number | 房间 ID（0-3），决定 RTP 和倍率范围 |

**只读配置字段**（由服务端硬编码）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `maxSlots` | number | 最大投注槽位数（3） |
| `bettingDuration` | number | 投注阶段持续时间（5 秒） |
| `waitingDuration` | number | 等待阶段持续时间（3 秒） |
| `minMultiplier` | number | 最小倍率（由房间决定） |
| `maxMultiplier` | number | 最大倍率（由房间决定） |

## 游戏阶段

| 阶段 | 说明 | 持续时间 |
|------|------|----------|
| betting | 接受投注 | 5-8 秒 |
| flying | 倍率上升，可兑现 | 随机 |
| waiting | 显示结果 | 3-5 秒 |

## placeBet

在 betting 阶段下注（单个槽位）。如需多槽位下注，需多次调用。

```javascript
const result = await centrifuge.rpc('placeBet', {
    gameId: 'inhousegame:crash',
    amount: '100.00',
    clientSeed: 'player_seed_123',
    slotId: 1
});
```

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `gameId` | string | 是 | 固定为 `inhousegame:crash` |
| `amount` | string | 是 | 投注金额 |
| `clientSeed` | string | 是 | 客户端种子 |
| `slotId` | number | 是 | 槽位 ID（0-2） |

**响应**：

```json
{
    "roundId": "123456789012345678",
    "slotId": 1,
    "amount": "100.00",
    "status": "active"
}
```

## cashOut

在 flying 阶段兑现指定槽位。

```javascript
const result = await centrifuge.rpc('cashOut', {
    gameId: 'inhousegame:crash',
    slotId: 1
});
```

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `gameId` | string | 是 | 固定为 `inhousegame:crash` |
| `slotId` | number | 是 | 要兑现的槽位 ID |

**响应**：

```json
{
    "results": [{
        "slotId": 1,
        "betAmount": "100.00",
        "multiplier": 2.35,
        "payout": "235.00"
    }]
}
```

## cancelBet

在 betting 阶段取消投注。

```javascript
const result = await centrifuge.rpc('cancelBet', {
    gameId: 'inhousegame:crash',
    slotId: 1
});
```

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `gameId` | string | 是 | 固定为 `inhousegame:crash` |
| `slotId` | number | 是 | 要取消的槽位 ID |

**响应**：

```json
{
    "slotId": 1,
    "refundAmount": "100.00"
}
```

## getState

获取当前游戏状态和用户投注信息。

```javascript
const result = await centrifuge.rpc('getState', {
    gameId: 'inhousegame:crash'
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "phase": "flying",
    "currentMultiplier": 1.85,
    "elapsedTime": 3500,
    "timeRemaining": 0,
    "totalPlayers": 45,
    "totalBets": "15230.00",
    "myBets": [
        { "slotId": 1, "amount": "100.00", "status": "active" },
        { "slotId": 2, "amount": "50.00", "status": "active" }
    ],
    "nextRound": {
        "roundId": "123456789012345679",
        "startsIn": 5000
    }
}
```

**响应字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 当前回合 ID |
| `phase` | string | 当前阶段：`betting`/`flying`/`waiting` |
| `currentMultiplier` | number | 当前倍率（flying 阶段实时更新） |
| `elapsedTime` | number | 当前阶段已经过时间（毫秒） |
| `timeRemaining` | number | 当前阶段剩余时间（毫秒），flying 阶段为 0 |
| `totalPlayers` | number | 当前回合参与玩家数 |
| `totalBets` | string | 当前回合总投注金额 |
| `myBets` | array | 用户在当前回合的投注列表 |
| `nextRound` | object | 下一轮信息（可选） |

**投注状态（status）**：

| 状态 | 说明 |
|------|------|
| `active` | 进行中 |
| `won` | 已兑现获胜 |
| `lost` | 已 Crash 失败 |
| `cancelled` | 已取消 |

## 订阅初始数据

订阅频道 `game:crash:room:{roomId}` 时，服务器会在订阅响应中推送当前房间状态快照：

```javascript
const sub = centrifuge.newSubscription('game:crash:room:0');
sub.on('subscribed', (ctx) => {
    const data = ctx.data;  // 初始数据
    console.log('当前回合:', data.roundId);
    console.log('当前阶段:', data.phase);
    console.log('当前玩家:', data.players);
});
sub.subscribe();
```

**初始数据结构**：

```json
{
    "roundId": "123456789012345678",
    "phase": "CRASH_PHASE_FLYING",
    "currentMultiplier": 1.85,
    "phaseStartTime": 1705900000000,
    "phaseEndTime": 1705900005000,
    "players": [
        {
            "nickname": "Pl***23",
            "amount": "100.00",
            "currency": "USDT",
            "status": "active"
        }
    ],
    "totalPlayers": 45,
    "totalBets": "15230.00"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 当前回合 ID |
| `phase` | string | 当前阶段枚举值 |
| `currentMultiplier` | number | 当前倍率（flying 阶段有效） |
| `phaseStartTime` | number | 阶段开始时间（Unix 毫秒） |
| `phaseEndTime` | number | 阶段结束时间（Unix 毫秒） |
| `players` | array | 当前回合玩家列表（昵称已脱敏） |
| `totalPlayers` | number | 参与玩家总数 |
| `totalBets` | string | 总投注金额 |

## 广播事件

订阅频道 `game:crash:room:{roomId}` 接收以下事件（roomId 为 0-3）。

前端根据消息字段组合区分事件类型：
- 有 `phase` + `nextPhase` → 阶段变更事件
- 有 `multiplier` + `elapsedTime` → 倍率更新事件
- 有 `crashPoint` → 回合结束事件
- 有 `nickname` + `amount`（无 `multiplier`）→ 玩家投注事件
- 有 `nickname` + `amount` + `currency`（无 `payout`）→ 玩家取消事件
- 有 `nickname` + `multiplier` + `payout` → 玩家兑现事件

### 阶段变更事件

```json
{
    "roundId": "123456789012345678",
    "phase": "CRASH_PHASE_BETTING",
    "duration": 5000,
    "nextPhase": "CRASH_PHASE_FLYING"
}
```

**phase 枚举值**：`CRASH_PHASE_BETTING`、`CRASH_PHASE_FLYING`、`CRASH_PHASE_WAITING`

### 倍率更新事件

flying 阶段持续推送（20fps）：

```json
{
    "multiplier": 1.85,
    "elapsedTime": 3500
}
```

### 回合结束事件

```json
{
    "roundId": "123456789012345678",
    "crashPoint": 3.45
}
```

### 玩家投注事件

有玩家下注时广播：

```json
{
    "nickname": "Player123",
    "amount": "100.00",
    "currency": "USDT"
}
```

### 玩家取消投注事件

有玩家取消投注时广播：

```json
{
    "nickname": "Player123",
    "amount": "100.00",
    "currency": "USDT"
}
```

### 玩家兑现事件

有玩家兑现时广播：

```json
{
    "nickname": "Player123",
    "multiplier": 2.35,
    "payout": "235.00",
    "currency": "USDT"
}
```

## 赔率计算

```
赔率 = 兑现时的倍率
收益 = 投注金额 × 倍率
```

倍率从 1.00x 开始，按指数曲线上升直到随机 Crash 点。

## 公平性验证

`POST /v1/fairness/crash/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "crashPoint": 3.45 }`
