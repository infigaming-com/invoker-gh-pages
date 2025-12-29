# Plinko 游戏 API

球从顶部落下，经过钉子随机弹跳落入槽位。支持 8-16 行，三种难度（low/medium/high）。采用延迟结算机制。

## plinko.placeBet

```javascript
const result = await centrifuge.rpc('plinko.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',               // "0" 为试玩
    params: {
        rows: 12,                  // 行数 8-16
        difficulty: 'medium'       // low/medium/high
    }
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

**注意**：响应不包含 `balance`，需在动画播放完成后调用 `settleBet` 获取余额。

## plinko.settleBet

动画播放完成后调用此接口完成余额结算。

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
