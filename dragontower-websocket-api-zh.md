# DragonTower 游戏 API

爬塔游戏：每层闯关成功倍率递增，最多 9 层，可随时提现。支持五种难度。

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
    "status": "playing",
    "currentLayer": 0,
    "maxLayers": 9,
    "difficulty": "medium",
    "currentMultiplier": "1.00",
    "nextMultiplier": "1.47",
    "nextProbability": 0.6667,
    "completedLayers": [],
    "balance": "990.00000000"
}
```

## dragontower.climb

闯关下一层。

```javascript
const result = await centrifuge.rpc('dragontower.climb', {
    roundId: '123456789012345678'
});
```

**响应（成功）**：

```json
{
    "survived": true,
    "status": "playing",
    "currentLayer": 1,
    "currentMultiplier": "1.47",
    "nextMultiplier": "2.21",
    "nextProbability": 0.6667,
    "completedLayers": [0]
}
```

**响应（失败）**：

```json
{
    "survived": false,
    "status": "finished",
    "currentMultiplier": "0.00",
    "balance": "990.00000000"
}
```

## dragontower.cashOut

提现当前赢利（至少完成一层）。

```javascript
const result = await centrifuge.rpc('dragontower.cashOut', {
    roundId: '123456789012345678'
});
```

**响应**：

```json
{
    "status": "cashed_out",
    "payout": "33.10000000",
    "currentMultiplier": "3.31",
    "completedLayers": [0, 1, 2],
    "balance": "1023.10000000"
}
```

## dragontower.checkActive / dragontower.resume

```javascript
// 检查活跃游戏
const active = await centrifuge.rpc('dragontower.checkActive', {});

// 恢复游戏
const state = await centrifuge.rpc('dragontower.resume', {
    roundId: '123456789012345678'
});
```

## 难度与倍率

| 难度 | 每层成功率 | 最大倍率（9层） |
|------|-----------|----------------|
| Easy | 75% | 7.51x |
| Medium | 66.7% | 19.68x |
| Hard | 50% | 512x |
| Expert | 33.3% | 19,683x |
| Master | 25% | 262,144x |

**Medium 难度倍率示例**：

| 层数 | 1 | 2 | 3 | 4 | 5 | 9 |
|-----|-----|-----|-----|-----|-----|------|
| 倍率 | 1.47x | 2.21x | 3.31x | 4.97x | 7.45x | 19.68x |

## 公平性验证

`POST /v1/fairness/dragontower/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1,
    "difficulty": "medium"
}
```

返回：`{ "layerResults": [true, true, false, ...], "multipliers": [1.47, 2.21, ...] }`
