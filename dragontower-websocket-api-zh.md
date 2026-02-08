# Dragon Tower 游戏 API

龙塔攀登：逐层向上攀登避开龙的位置，每成功一层倍率递增，可随时提现。支持五种难度，最高 9 层。

## dragontower.placeBet

开始新游戏。

```javascript
const result = await centrifuge.rpc('dragontower.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                  // "0" 为试玩
    params: {
        difficulty: 'medium'          // easy/medium/hard/expert/master
    }
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
        "difficulty": "medium",
        "currentLayer": 0,
        "maxLayers": 9,
        "path": [],
        "currentMultiplier": "1.00000000",
        "nextMultiplier": "1.45230556"
    }
}
```

## dragontower.climb

攀登一层。

```javascript
const result = await centrifuge.rpc('dragontower.climb', {
    roundId: '123456789012345678'
});
```

**响应（生存）**：

```json
{
    "survived": true,
    "gameState": {
        "status": "playing",
        "currentLayer": 1,
        "maxLayers": 9,
        "path": [0],
        "currentMultiplier": "1.45230556",
        "nextMultiplier": "2.16633750"
    }
}
```

**响应（失败）**：

```json
{
    "survived": false,
    "gameState": {
        "status": "finished",
        "currentMultiplier": "0.00000000"
    },
    "gameResult": {
        "dragonPositions": [0],
        "layersCompleted": 0,
        "finalMultiplier": "0.00000000",
        "payout": "0.00000000"
    }
}
```

**响应（到达顶层自动结算）**：

```json
{
    "survived": true,
    "gameState": {
        "status": "finished",
        "currentLayer": 9,
        "path": [0, 1, 2, 3, 4, 5, 6, 7, 8]
    },
    "gameResult": {
        "dragonPositions": [0, 1, 2, 3, 4, 5, 6, 7, 8],
        "layersCompleted": 9,
        "finalMultiplier": "31.69702400",
        "payout": "316.97024000"
    }
}
```

## dragontower.cashOut

提现当前赢利（至少攀登一层）。

```javascript
const result = await centrifuge.rpc('dragontower.cashOut', {
    roundId: '123456789012345678'
});
```

**响应**：

```json
{
    "payout": "14.52305560",
    "gameState": {
        "status": "cashed_out",
        "currentLayer": 1,
        "path": [0],
        "currentMultiplier": "1.45230556",
        "nextMultiplier": "2.16633750"
    },
    "result": {
        "dragonPositions": [0],
        "layersCompleted": 1,
        "finalMultiplier": "1.45230556",
        "payout": "14.52305560"
    }
}
```

## dragontower.autoPlay

自动攀登到指定目标层数（下注 + 攀登一次性完成）。

```javascript
const result = await centrifuge.rpc('dragontower.autoPlay', {
    betRequest: {
        clientSeed: 'player_seed_123',
        amount: '10.00',
        params: {
            difficulty: 'medium'
        }
    },
    targetLayers: 5                   // 1-9
});
```

**响应（成功到达目标层）**：

```json
{
    "roundId": "123456789012345678",
    "survived": true,
    "layersCompleted": 5,
    "payout": "70.24397000",
    "gameResult": {
        "dragonPositions": [0, 1, 2, 3, 4],
        "layersCompleted": 5,
        "finalMultiplier": "7.02439700",
        "payout": "70.24397000"
    }
}
```

**响应（中途失败）**：

```json
{
    "roundId": "123456789012345678",
    "survived": false,
    "layersCompleted": 3,
    "payout": "0.00000000",
    "gameResult": {
        "dragonPositions": [0, 1, 2],
        "layersCompleted": 3,
        "finalMultiplier": "0.00000000",
        "payout": "0.00000000"
    }
}
```

## dragontower.checkActive

```javascript
const active = await centrifuge.rpc('dragontower.checkActive', {});
// { hasActiveGame: true, roundId: "...", gameState: {...} }
```

## 难度配置

| 难度 | 列数 | 成功率 | 特点 |
|------|------|--------|------|
| Easy | 4 | 75% (3/4) | 低风险，倍率增长缓慢 |
| Medium | 3 | 66.67% (2/3) | 中等风险，默认难度 |
| Hard | 2 | 50% (1/2) | 高风险，倍率增长快 |
| Expert | 3 | 33.33% (1/3) | 极高风险 |
| Master | 4 | 25% (1/4) | 最高风险，倍率增长极快 |

**Medium 难度倍率示例**：

| 层数 | 1 | 2 | 3 | 5 | 7 | 9 |
|-----|------|------|------|------|-------|-------|
| 倍率 | 1.45x | 2.17x | 3.22x | 7.02x | 15.07x | 31.70x |

## 赔率计算

```
有效RTP(n) = RTP × (1 - greedPenalty × (n / maxLayers)²)
倍率(n) = 有效RTP(n) / 成功率^n
```

- **RTP**: 97%（配置可调）
- **greedPenalty**: 0.15（层数越高，有效 RTP 越低）
- **倍率精度**: 8 位小数

## 公平性验证

DragonTower 目前未提供独立的公平性验证 REST API。游戏结果可通过种子信息在客户端独立验证：

```
对于每一层 i（i = 0 到 maxLayers-1）：
  1. 组合种子：seedStr = "serverSeed:clientSeed:nonce:i"
  2. 计算哈希：hash = SHA256(seedStr)
  3. 取前 8 位十六进制：hashFirst8 = hash[:8]
  4. 转换为随机数：randomValue = parseInt(hashFirst8, 16) / 0x100000000
  5. 判断结果：survived = randomValue < successRate
```
