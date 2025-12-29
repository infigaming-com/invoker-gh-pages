# Mines 游戏 API

扫雷游戏：在网格中揭开安全格子避免触雷，每揭开一个格子赔率递增，可随时提现。支持 3x3/5x5/7x7 网格。

## mines.placeBet

开始新游戏。

```javascript
const result = await centrifuge.rpc('mines.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                  // "0" 为试玩
    params: {
        minesCount: 5,                // 地雷数量
        gridType: '5x5'               // 3x3/5x5/7x7
    }
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "balance": "990.00000000",
    "gameState": {
        "status": "playing",
        "betAmount": "10.00000000",
        "minesCount": 5,
        "gridType": "5x5",
        "revealedTiles": [],
        "safeTilesRevealed": 0
    }
}
```

## mines.revealTile

揭开一个格子。

```javascript
const result = await centrifuge.rpc('mines.revealTile', {
    roundId: '123456789012345678',
    tileIndex: 12                     // 格子索引：0-8(3x3)/0-24(5x5)/0-48(7x7)
});
```

**响应（安全）**：

```json
{
    "isMine": false,
    "gameState": {
        "status": "playing",
        "revealedTiles": [12],
        "safeTilesRevealed": 1
    },
    "currentMultiplier": "1.04",
    "nextMultiplier": "1.09"
}
```

**响应（触雷）**：

```json
{
    "isMine": true,
    "gameState": { "status": "finished" },
    "result": {
        "minePositions": [3, 7, 15, 18, 22],
        "finalMultiplier": "0",
        "payout": "0"
    }
}
```

## mines.cashOut

提现当前赢利（至少揭开一个安全格子）。

```javascript
const result = await centrifuge.rpc('mines.cashOut', {
    roundId: '123456789012345678'
});
```

**响应**：

```json
{
    "payout": "15.60",
    "gameState": { "status": "finished" },
    "result": {
        "minePositions": [3, 7, 15, 18, 22],
        "finalMultiplier": "1.56",
        "payout": "15.60"
    },
    "balance": "1005.60"
}
```

## mines.checkActive / mines.resume

```javascript
// 检查活跃游戏
const active = await centrifuge.rpc('mines.checkActive', {});
// { hasActiveGame: true, roundId: "...", gameState: {...} }

// 恢复游戏
const state = await centrifuge.rpc('mines.resume', {
    roundId: '123456789012345678'     // 可选
});
```

## 赔率计算

```
基础赔率 = (总格子数 / (总格子数 - 地雷数)) ^ 揭开格子数
最终赔率 = 基础赔率 × 0.99
```

**示例（5x5 网格，5 个地雷）**：

| 揭开数 | 1 | 2 | 5 | 10 | 15 | 20 |
|-------|-----|-----|-----|------|------|------|
| 赔率 | 1.04x | 1.09x | 1.26x | 1.65x | 2.38x | 4.50x |

## 公平性验证

`POST /v1/fairness/mines/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1,
    "minesCount": 5,
    "gridType": "5x5"
}
```

返回：`{ "minePositions": [3, 7, 15, 18, 22] }`
