# Dice 游戏 API

预测骰子结果是否大于或小于目标值（4-96），RTP 99%。

## dice.placeBet

```javascript
const result = await centrifuge.rpc('dice.placeBet', {
    clientSeed: 'player_seed_123',  // 客户端种子
    amount: '10.00',                 // 投注金额，"0" 为试玩
    params: {
        target: 50,                  // 目标数字 (4-96)
        isRollOver: true             // true=大于, false=小于
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
            "winAmount": "9.80000000",
            "isWin": true,
            "multiplier": "1.98000000",
            "diceOutcome": {
                "roll": "75.23",
                "target": "50.00",
                "isRollOver": true
            }
        }
    }
}
```

## 赔率计算

- **Roll Over**：`99 / (99.99 - target)`
- **Roll Under**：`99 / target`

## 公平性验证

`POST /v1/fairness/dice/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "roll": 53.42857142 }`
