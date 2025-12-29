# Dragon Tiger 游戏 API

龙虎斗：预测龙或虎谁的牌点数更大（1-13），或预测平局。RTP 97%。

## dragontiger.placeBet

```javascript
const result = await centrifuge.rpc('dragontiger.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                 // "0" 为试玩
    params: {
        dragonBet: '10.00',          // 龙投注
        tigerBet: '0',               // 虎投注（不能与龙同时投注）
        tieBet: '0'                  // 和投注
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
            "winAmount": "9.70000000",
            "isWin": true,
            "multiplier": "0.97000000",
            "dragonTigerOutcome": {
                "dragonCard": 10,
                "tigerCard": 5,
                "result": "dragon",
                "dragonPayout": "9.70000000",
                "tigerPayout": "0.00000000",
                "tiePayout": "0.00000000"
            }
        }
    }
}
```

## 赔率规则

| 投注类型 | 赔率 | 说明 |
|---------|------|------|
| 龙/虎 | 0.97:1 | 应用 97% RTP |
| 和 | 8:1 | 无 RTP 调整 |
| 平局退款 | 50% × 0.97 | 龙/虎投注退回部分 |

- **龙/虎不能同时投注**
- **卡牌点数**：1(A) - 13(K)

## 公平性验证

`POST /v1/fairness/dragontiger/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "dragonCard": 10, "tigerCard": 5, "result": "dragon" }`
