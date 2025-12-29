# Keno 游戏 API

从 1-40 中选择 1-10 个数字，系统开出 10 个号码，根据匹配数量获得派彩。支持 4 种难度模式。

## keno.placeBet

```javascript
const result = await centrifuge.rpc('keno.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                      // "0" 为试玩
    params: {
        selectedNumbers: [3, 7, 15, 22, 28],  // 1-10 个数字（1-40 范围）
        difficulty: 'classic'                  // low/classic/medium/high
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
            "winAmount": "25.00000000",
            "isWin": true,
            "multiplier": "2.50000000",
            "kenoOutcome": {
                "selectedNumbers": [3, 7, 15, 22, 28],
                "drawnNumbers": [3, 7, 9, 12, 15, 18, 22, 25, 31, 35],
                "matchedNumbers": [3, 7, 15, 22],
                "matchCount": 4,
                "difficulty": "classic"
            }
        }
    }
}
```

## 赔率表（Classic 难度）

| 选数 | 0中 | 1中 | 2中 | 3中 | 4中 | 5中 | 6中 | 7中 | 8中 | 9中 | 10中 |
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|------|
| 1 | 0 | 3.96 | - | - | - | - | - | - | - | - | - |
| 2 | 0 | 1.9 | 4.5 | - | - | - | - | - | - | - | - |
| 3 | 0 | 1 | 3.1 | 10.4 | - | - | - | - | - | - | - |
| 4 | 0 | 0.8 | 1.8 | 5 | 22.5 | - | - | - | - | - | - |
| 5 | 0 | 0.25 | 1.4 | 4.1 | 16.5 | 36 | - | - | - | - | - |
| 10 | 0 | 0 | 0 | 0 | 1.4 | 2.25 | 4.5 | 8 | 17 | 50 | 100 |

完整赔率表请查看 [Keno 详细设计](./keno-detailed-design-zh.md)。

## 公平性验证

`POST /v1/fairness/keno/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1,
    "selectedNumbers": [3, 7, 15, 22, 28],
    "difficulty": "classic"
}
```

返回：`{ "drawnNumbers": [...], "matchedNumbers": [...], "matchCount": 4, "multiplier": 2.5 }`
