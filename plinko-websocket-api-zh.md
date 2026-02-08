# Plinko 游戏 API

球从顶部落下，经过钉子随机弹跳落入槽位。支持 8-16 行，三种难度（low/medium/high）。支持客户端控制延迟结算（详见[通用文档 - 延迟结算](./common-websocket-api-zh.md#5-即时游戏延迟结算)）。

## plinko.placeBet

```javascript
// 延迟结算模式（默认，播放动画后调 settleBet）
const result = await centrifuge.rpc('plinko.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',               // "0" 为试玩
    params: {
        rows: 12,                  // 行数 8-16
        difficulty: 'medium'       // low/medium/high
    }
});

// 立即结算模式（跳过动画）
const result = await centrifuge.rpc('plinko.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',
    immediateSettlement: true,     // 强制立即结算
    params: { rows: 12, difficulty: 'medium' }
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "outcome": {
        "gameResult": {
            "betAmount": "10.00000000",
            "winAmount": "18.00000000",
            "isWin": true,
            "multiplier": "1.80000000",
            "plinkoOutcome": {
                "rows": 12,
                "difficulty": "medium",
                "path": [0, 1, 1, 0, 1, 0, 0, 1, 1, 0, 1, 0],
                "finalSlot": 6,
                "multiplier": "1.80000000"
            }
        }
    },
    "hasAnimation": true
}
```

- 延迟结算模式（默认）：响应不包含 `balance`，需调用 `settleBet`
- 立即结算模式（`immediateSettlement: true`）：立即结算，无需调用 `settleBet`

## plinko.settleBet

仅在延迟结算模式下需要调用，动画播放完成后完成余额结算。

```javascript
const result = await centrifuge.rpc('plinko.settleBet', {
    roundId: '123456789012345678'
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "balance": "1018.00"
}
```

- **幂等性**：重复调用不会重复结算
- **超时自动结算**：20 秒内未调用会自动结算

## 赔率表示例（12 行 Medium）

| 槽位 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|-----|---|---|---|---|---|---|---|---|---|---|-----|-----|-----|
| 赔率 | 24 | 12 | 4 | 2 | 1.2 | 0.6 | 0.4 | 0.6 | 1.2 | 2 | 4 | 12 | 24 |

完整赔率表请查看 [Plinko 详细设计](./plinko-detailed-design-zh.md)。

## 公平性验证

`POST /v1/fairness/plinko/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1,
    "rows": 12,
    "difficulty": "medium"
}
```

返回：`{ "path": [0,1,1,...], "finalSlot": 7, "multiplier": 5.6 }`
