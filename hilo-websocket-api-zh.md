# HiLo 游戏 API

高低牌游戏：预测下一张牌比当前牌高还是低。每次猜对倍率累积，可随时提现。支持跳过换牌。

## hilo.placeBet

开始新游戏。

```javascript
const result = await centrifuge.rpc('hilo.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00'                   // "0" 为试玩
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "gameState": {
        "roundId": "123456789012345678",
        "status": "playing",
        "betAmount": "10.00000000",
        "currentRound": 0,
        "currentCard": 7,
        "cardHistory": [7],
        "currentMultiplier": "1.00000000",
        "nextMultiplier": "1.78290000",
        "remainingSkips": 50
    }
}
```

## hilo.choice

预测下一张牌。

```javascript
const result = await centrifuge.rpc('hilo.choice', {
    roundId: '123456789012345678',
    choice: 'higher'                  // higher/lower/skip
});
```

- **higher**：预测下张牌 >= 当前牌
- **lower**：预测下张牌 <= 当前牌
- **skip**：跳过换牌（不影响倍率，最多 50 次）

**响应（猜对）**：

```json
{
    "guessCorrect": true,
    "gameState": {
        "status": "playing",
        "currentRound": 1,
        "currentCard": 10,
        "cardHistory": [7, 10],
        "currentMultiplier": "1.78290000",
        "nextMultiplier": "5.56530000",
        "remainingSkips": 50
    }
}
```

**响应（猜错）**：

```json
{
    "guessCorrect": false,
    "gameState": {
        "status": "finished",
        "currentCard": 3,
        "cardHistory": [7, 3],
        "currentMultiplier": "0.00000000"
    },
    "gameResult": {
        "cardSequence": [7, 3],
        "roundsPlayed": 1,
        "finalMultiplier": "0.00000000",
        "payout": "0.00000000"
    }
}
```

**响应（跳过）**：

```json
{
    "guessCorrect": true,
    "gameState": {
        "status": "playing",
        "currentRound": 0,
        "currentCard": 11,
        "cardHistory": [7, 11],
        "currentMultiplier": "1.00000000",
        "nextMultiplier": "1.13450000",
        "remainingSkips": 49
    }
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
    "payout": "17.82900000",
    "gameState": {
        "status": "cashed_out",
        "currentRound": 1,
        "currentCard": 10,
        "cardHistory": [7, 10],
        "currentMultiplier": "1.78290000",
        "nextMultiplier": "0.00000000",
        "remainingSkips": 50
    },
    "result": {
        "cardSequence": [7, 10],
        "roundsPlayed": 1,
        "finalMultiplier": "1.78290000",
        "payout": "17.82900000"
    }
}
```

## hilo.checkActive

```javascript
const active = await centrifuge.rpc('hilo.checkActive', {});
// { hasActiveGame: true, roundId: "...", gameState: {...} }
```

## 赔率计算

```
Higher 概率 = (14 - 当前牌) / 13
Lower 概率 = 当前牌 / 13
单步倍率 = RTP / 概率（保留 4 位小数）
累积倍率 = 各步倍率相乘
```

- **RTP**: 96%（配置可调）
- **判定规则**: Higher = 下张牌 **>=** 当前牌, Lower = 下张牌 **<=** 当前牌（包含相等）
- **牌面值**: A=1, 2-10, J=11, Q=12, K=13

| 当前牌 | Higher 概率 | Higher 倍率 | Lower 概率 | Lower 倍率 |
|--------|------------|------------|-----------|-----------|
| A (1) | 100% (13/13) | 0.96x | 7.7% (1/13) | 12.48x |
| 4 | 76.9% (10/13) | 1.25x | 30.8% (4/13) | 3.12x |
| 7 | 53.8% (7/13) | 1.78x | 53.8% (7/13) | 1.78x |
| 10 | 30.8% (4/13) | 3.12x | 76.9% (10/13) | 1.25x |
| K (13) | 7.7% (1/13) | 12.48x | 100% (13/13) | 0.96x |

**注意**：A 选 Higher 和 K 选 Lower 的倍率为 0.96x（必赢但亏损），不存在"无法选择"的情况。

## 公平性验证

HiLo 目前未提供独立的公平性验证 REST API。游戏结果可通过种子信息在客户端独立验证：

```
对于每张牌 i（i = 0, 1, 2, ...）：
  1. 组合种子：seedStr = "serverSeed:clientSeed:nonce:i"
  2. 计算哈希：hash = SHA256(seedStr)
  3. 取前 8 位十六进制：hashFirst8 = hash[:8]
  4. 转换为整数：hashValue = parseInt(hashFirst8, 16)
  5. 计算牌值：card = (hashValue % 13) + 1
```
