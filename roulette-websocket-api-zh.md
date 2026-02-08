# Roulette 游戏 API

欧式轮盘：投注一个或多个位置，轮盘旋转后根据中奖号码（0-36）结算。支持 13 种投注类型，RTP 96%。

## roulette.placeBet

下注并立即出结果，支持单次请求多个投注。

```javascript
const result = await centrifuge.rpc('roulette.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                 // 总投注金额，"0" 为试玩
    params: {
        bets: [
            { type: 'straight', numbers: [17], amount: '5.00' },
            { type: 'red', amount: '5.00' }
        ]
    }
});
```

**响应（有中奖投注）**：

```json
{
    "roundId": "123456789012345678",
    "outcome": {
        "roundId": "123456789012345678",
        "gameResult": {
            "betAmount": "10.00000000",
            "winAmount": "4.86666667",
            "isWin": true,
            "multiplier": "1.48666667",
            "rouletteOutcome": {
                "winningNumber": 3,
                "color": "red",
                "winningBets": [
                    {
                        "betType": "red",
                        "stake": "5.00000000",
                        "multiplier": 1.9733333333333334,
                        "payout": "9.86666667"
                    }
                ],
                "totalStake": "10.00000000"
            }
        },
        "hasAnimation": false
    }
}
```

**响应（未中奖）**：

```json
{
    "roundId": "123456789012345678",
    "outcome": {
        "roundId": "123456789012345678",
        "gameResult": {
            "betAmount": "10.00000000",
            "winAmount": "-10.00000000",
            "isWin": false,
            "multiplier": "0.00000000",
            "rouletteOutcome": {
                "winningNumber": 0,
                "color": "green",
                "winningBets": [],
                "totalStake": "10.00000000"
            }
        },
        "hasAnimation": false
    }
}
```

## 投注类型

| 类型 | 说明 | numbers 参数 | 覆盖数 | 倍率 |
|------|------|-------------|--------|------|
| `straight` | 单号投注 | `[17]`（0-36） | 1 | 35.52x |
| `split` | 两号投注 | `[17, 18]` | 2 | 17.76x |
| `street` | 三号投注 | `[1, 2, 3]` | 3 | 11.84x |
| `corner` | 四号投注 | `[1, 2, 4, 5]` | 4 | 8.88x |
| `sixline` | 六号投注 | `[1, 2, 3, 4, 5, 6]` | 6 | 5.92x |
| `column` | 列投注 | `[1]`（1-3） | 12 | 2.96x |
| `dozen` | 打投注 | `[1]`（1-3） | 12 | 2.96x |
| `red` | 红色 | 不需要 | 18 | 1.97x |
| `black` | 黑色 | 不需要 | 18 | 1.97x |
| `odd` | 奇数 | 不需要 | 18 | 1.97x |
| `even` | 偶数 | 不需要 | 18 | 1.97x |
| `low` | 小（1-18） | 不需要 | 18 | 1.97x |
| `high` | 大（19-36） | 不需要 | 18 | 1.97x |

**numbers 参数说明**：
- `column`：`[1]` = 第一列（1,4,7...34），`[2]` = 第二列（2,5,8...35），`[3]` = 第三列（3,6,9...36）
- `dozen`：`[1]` = 1-12，`[2]` = 13-24，`[3]` = 25-36
- 外围投注（red/black/odd/even/low/high）不需要传 numbers

## 赔率计算

```
倍率 = (总数字数 / 覆盖数) × RTP
     = (37 / coverage) × 0.96
```

- **RTP**: 96%（配置可调）
- **总数字数**: 37（0-36，欧式轮盘）
- **0 号**: 绿色，所有外围投注均不覆盖 0

## 公平性验证

`POST /v1/fairness/roulette/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "winningNumber": 17, "color": "black" }`
