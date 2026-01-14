# Crash 游戏 API

多人实时倍率游戏：倍率从 1.00x 开始上升，玩家在 Crash 前兑现获得当前倍率的赔付。支持最多 3 个投注槽位。

## 获取游戏配置

使用通用接口 `common.getConfig` 获取 Crash 游戏配置：

```javascript
const result = await centrifuge.rpc('common.getConfig', {
    gameId: 'inhousegame:crash'
});
```

**响应**：

```json
{
    "gameId": "inhousegame:crash",
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
            "maxSlots": 3,
            "bettingDuration": 5,
            "waitingDuration": 3,
            "maxMultiplier": 100
        }
    }
}
```

**gameParameters 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `maxSlots` | number | 最大投注槽位数 |
| `bettingDuration` | number | 投注阶段持续时间（秒） |
| `waitingDuration` | number | 等待阶段持续时间（秒） |
| `maxMultiplier` | number | 最大倍率限制 |

## 游戏阶段

| 阶段 | 说明 | 持续时间 |
|------|------|----------|
| betting | 接受投注 | 5-8 秒 |
| flying | 倍率上升，可兑现 | 随机 |
| waiting | 显示结果 | 3-5 秒 |

## crash.placeBet

在 betting 阶段下注。

```javascript
const result = await centrifuge.rpc('crash.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '150.00',                     // 总金额（可选）
    params: {
        slots: [
            { slotId: 1, amount: '100.00' },
            { slotId: 2, amount: '50.00' }
        ]
    }
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "slots": [
        { "slotId": 1, "amount": "100.00", "status": "active" },
        { "slotId": 2, "amount": "50.00", "status": "active" }
    ],
    "balance": "9850.00"
}
```

## crash.cashOut

在 flying 阶段兑现。

```javascript
const result = await centrifuge.rpc('crash.cashOut', {
    roundId: '123456789012345678',
    slotId: 1,                            // 指定槽位，0 = 全部兑现
    cashOutAll: false
});
```

**响应**：

```json
{
    "results": [{
        "slotId": 1,
        "betAmount": "100.00",
        "multiplier": 2.35,
        "winAmount": "235.00",
        "profit": "135.00"
    }],
    "balance": "10085.00"
}
```

## crash.cancelBet

在 betting 阶段取消投注。

```javascript
const result = await centrifuge.rpc('crash.cancelBet', {
    roundId: '123456789012345678',
    slotId: 1                             // 0 = 取消全部
});
```

## crash.getState

获取当前游戏状态和用户投注信息。

```javascript
const result = await centrifuge.rpc('crash.getState', {});
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
        { "slotId": 1, "amount": "100.00", "status": "active" }
    ],
    "pendingBets": [
        { "slotId": 2, "amount": "50.00", "status": "pending" }
    ],
    "nextRound": {
        "roundId": "123456789012345679",
        "startsIn": 3000
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
| `pendingBets` | array | 用户预约到下一轮的投注列表 |
| `nextRound` | object | 下一轮信息（可选） |

**投注状态（status）**：

| 状态 | 说明 |
|------|------|
| `pending` | 待处理（预约下一轮） |
| `active` | 进行中 |
| `won` | 已兑现获胜 |
| `lost` | 已 Crash 失败 |
| `cancelled` | 已取消 |

## 广播事件

连接后自动接收以下事件：

### CRASH_PHASE_CHANGE

```json
{
    "roundId": "123456789012345678",
    "phase": "betting",
    "duration": 5000,
    "nextPhase": "flying"
}
```

### CRASH_MULTIPLIER_UPDATE

flying 阶段持续推送（20fps）：

```json
{
    "multiplier": 1.85,
    "elapsedTime": 3500
}
```

### CRASH_ROUND_END

```json
{
    "roundId": "123456789012345678",
    "crashPoint": 3.45,
    "duration": 8523,
    "winners": [{
        "playerId": "player123",
        "betAmount": "100.00",
        "cashOutMultiplier": 2.35,
        "winAmount": "235.00"
    }],
    "totalBets": "15430.00",
    "totalPayout": "8235.00",
    "nextRoundIn": 3000
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
