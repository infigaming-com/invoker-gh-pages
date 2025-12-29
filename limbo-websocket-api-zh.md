# Limbo 游戏 API

设定目标倍率（1.01-10000），系统生成随机倍率。若结果 ≥ 目标倍率则获胜，派彩 = 投注额 × 目标倍率。RTP 99%。

## limbo.placeBet

```javascript
const result = await centrifuge.rpc('limbo.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                  // "0" 为试玩
    params: {
        targetMultiplier: '2.00'      // 目标倍率 1.01-10000
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
            "winAmount": "20.00000000",
            "isWin": true,
            "multiplier": "2.00000000",
            "limboOutcome": {
                "resultMultiplier": "2.35",
                "targetMultiplier": "2.00"
            }
        }
    }
}
```

## 赔率计算

- **胜率**：`99% / targetMultiplier`
- **派彩**：`betAmount × targetMultiplier`（获胜时）

| 目标倍率 | 胜率 |
|---------|------|
| 1.01x | 98.02% |
| 2.00x | 49.50% |
| 10.00x | 9.90% |
| 100.00x | 0.99% |

## 公平性验证

`POST /v1/fairness/limbo/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "resultMultiplier": 2.35 }`
