# Crash 游戏 API

多人实时倍率游戏：倍率从 1.00x 开始上升，玩家在 Crash 前兑现获得当前倍率的赔付。支持最多 3 个投注槽位。

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

获取当前游戏状态。

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
    "myBets": [{ "slotId": 1, "amount": "100.00", "status": "active" }],
    "totalPlayers": 45,
    "totalBets": "15230.00"
}
```

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
