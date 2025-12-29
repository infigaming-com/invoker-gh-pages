# HiLo 游戏 API

高低牌游戏：预测下一张牌比当前牌高（Higher）还是低（Lower）。每次猜对赔率累积，可随时提现。

## hilo.startGame

开始新游戏。

```javascript
const result = await centrifuge.rpc('hilo.startGame', {
    clientSeed: 'player_seed_123',
    amount: '10.00'                   // "0" 为试玩
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentCard": 7,
    "cardHistory": [7],
    "currentMultiplier": "1.0000",
    "canCashout": false,
    "higherProbability": 0.4615,
    "lowerProbability": 0.4615,
    "higherMultiplier": "2.1450",
    "lowerMultiplier": "2.1450"
}
```

## hilo.makeChoice

预测下一张牌。

```javascript
const result = await centrifuge.rpc('hilo.makeChoice', {
    roundId: '123456789012345678',
    choice: 'higher'                  // higher/lower/skip/same
});
```

- **higher**：预测下张牌更大（当前牌非 K）
- **lower**：预测下张牌更小（当前牌非 A）
- **skip**：跳过当前牌（仅未做任何猜测前可用）
- **same**：预测相同（边界情况自动应用）

**响应（猜对）**：

```json
{
    "guessCorrect": true,
    "status": "playing",
    "currentCard": 10,
    "cardHistory": [7, 10],
    "currentMultiplier": "2.1450",
    "canCashout": true,
    "higherMultiplier": "4.2893",
    "lowerMultiplier": "1.4298"
}
```

**响应（猜错）**：

```json
{
    "guessCorrect": false,
    "status": "finished",
    "currentCard": 3,
    "finalPayout": "0.00000000"
}
```

## hilo.cashOut

提现当前赢利（至少猜对一次）。

```javascript
const result = await centrifuge.rpc('hilo.cashOut', {
    roundId: '123456789012345678'
});
```

**响应**：

```json
{
    "status": "cashed_out",
    "payout": "21.45000000",
    "finalPayout": "21.45000000",
    "currentMultiplier": "2.1450"
}
```

## hilo.checkActive / hilo.resume

```javascript
// 检查活跃游戏
const active = await centrifuge.rpc('hilo.checkActive', {});

// 恢复游戏
const state = await centrifuge.rpc('hilo.resume', {
    roundId: '123456789012345678'     // 可选
});
```

## 赔率计算

```
赔率 = 0.99 / 获胜概率
```

**牌面值**：A=1, 2-10, J=11, Q=12, K=13

| 当前牌 | Higher 概率 | Lower 概率 | Higher 赔率 | Lower 赔率 |
|--------|------------|-----------|------------|-----------|
| A (1) | 92.3% | 0% | 1.07x | - |
| 7 | 46.2% | 46.2% | 2.14x | 2.14x |
| Q (12) | 7.7% | 84.6% | 12.86x | 1.17x |
| K (13) | 0% | 92.3% | - | 1.07x |

## 公平性验证

`POST /v1/fairness/hilo/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "cards": [7, 10, 3], "results": [true, false] }`
