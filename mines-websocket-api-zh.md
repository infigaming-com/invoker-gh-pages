# Mines 游戏 WebSocket API 文档

## 游戏概述

Mines 是一款经典的扫雷游戏，玩家在网格中揭开安全格子，避免触雷。每揭开一个安全格子，赔率递增，玩家可以随时提现或继续冒险。

## 游戏特性

- **会话型游戏**：支持多步操作，可随时提现
- **多种网格**：支持 3x3、5x5、7x7 网格
- **灵活地雷数**：可自定义地雷数量
- **自动提现**：5分钟无操作自动结算
- **断线重连**：支持游戏恢复功能
- **可证明公平**：完整的 Provably Fair 机制

## 1. PLACE_BET - 开始游戏

开始新的 Mines 游戏。每个玩家同时只能有一个活跃的 Mines 游戏。

### 请求消息

```json
{
  "i": "msg_009",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",  // 可选：空字符串""或"0"表示试玩模式
    "gameParams": {
      "minesParams": {
        "minesCount": 5,    // 地雷数量（1-24，取决于网格大小）
        "gridType": "5x5"   // 网格类型："3x3"、"5x5"、"7x7"
      }
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |
| `minesParams.minesCount` | number | 是 | 地雷数量，必须小于格子总数 |
| `minesParams.gridType` | string | 是 | 网格类型："3x3"(9格)、"5x5"(25格)、"7x7"(49格) |

### 响应消息

```json
{
  "i": "msg_009",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "balance": "990.00000000",
    "gameState": {
      "@type": "type.googleapis.com/api.game.v1.MinesGameState",
      "roundId": "123456789012345678",
      "status": "playing",
      "betAmount": "10.00000000",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [],
      "safeTilesRevealed": 0
    }
  }
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID |
| `balance` | string | 投注后的玩家余额（小数点后8位） |
| `gameState` | MinesGameState | 游戏状态对象 |
| `gameState.roundId` | string | 游戏回合ID |
| `gameState.status` | string | 游戏状态："playing"（进行中） |
| `gameState.betAmount` | string | 投注金额 |
| `gameState.minesCount` | number | 地雷数量 |
| `gameState.gridType` | string | 网格类型 |
| `gameState.revealedTiles` | number[] | 已揭示的格子索引（初始为空） |
| `gameState.safeTilesRevealed` | number | 已揭示的安全格子数量（初始为0） |

**注意**：
- 投注时不返回 `gameResult`，只有在游戏结束时（踩到地雷或提现）才返回
- `balance` 是从聚合器实时返回的最新余额，无需额外查询

## 2. MINES_REVEAL_TILE - 揭开格子

在进行中的游戏里揭开一个格子。

### 请求消息

```json
{
  "i": "msg_010",
  "t": "MINES_REVEAL_TILE",
  "p": {
    "roundId": "123456789012345678",
    "tileIndex": 12  // 格子索引：0-8(3x3)、0-24(5x5)、0-48(7x7)
  }
}
```

### 响应消息（安全格子）

```json
{
  "i": "msg_010",
  "t": "MINES_REVEAL_TILE",
  "p": {
    "isMine": false,
    "gameState": {
      "roundId": "123456789012345678",
      "status": "playing",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12],
      "safeTilesRevealed": 1
    },
    "currentMultiplier": "1.04",
    "nextMultiplier": "1.09"
  }
}
```

### 响应消息（触雷）

```json
{
  "i": "msg_010",
  "t": "MINES_REVEAL_TILE",
  "p": {
    "isMine": true,
    "gameState": {
      "roundId": "123456789012345678",
      "status": "finished",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 15],
      "safeTilesRevealed": 1
    },
    "result": {
      "minePositions": [3, 7, 15, 18, 22],
      "safeTilesRevealed": 1,
      "finalMultiplier": "0",
      "payout": "0",
      "provablyFair": {
        "serverSeed": "server_seed_revealed",
        "hashedServerSeed": "hash_abc123",
        "nonce": 1
      }
    }
  }
}
```

## 3. MINES_CASH_OUT - 结算提现

结算当前游戏并提现奖金。至少需要揭开一个安全格子才能提现。

### 请求消息

```json
{
  "i": "msg_011",
  "t": "MINES_CASH_OUT",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_011",
  "t": "MINES_CASH_OUT",
  "p": {
    "payout": "15.60",
    "gameState": {
      "roundId": "123456789012345678",
      "status": "finished",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 8, 20],
      "safeTilesRevealed": 3
    },
    "result": {
      "minePositions": [3, 7, 15, 18, 22],
      "safeTilesRevealed": 3,
      "finalMultiplier": "1.56",
      "payout": "15.60",
      "provablyFair": {
        "serverSeed": "server_seed_revealed",
        "hashedServerSeed": "hash_abc123",
        "nonce": 1
      }
    },
    "balance": "1005.60"
  }
}
```

## 4. MINES_GET_STATE - 获取游戏状态

获取当前游戏的详细状态。

### 请求消息

```json
{
  "i": "msg_012",
  "t": "MINES_GET_STATE",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_012",
  "t": "MINES_GET_STATE",
  "p": {
    "gameState": {
      "roundId": "123456789012345678",
      "status": "playing",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 8],
      "safeTilesRevealed": 2
    },
    "currentMultiplier": "1.09",
    "nextMultiplier": "1.14"
  }
}
```

## 5. MINES_CHECK_ACTIVE - 检查活跃游戏

检查玩家是否有未完成的游戏。

### 请求消息

```json
{
  "i": "msg_013",
  "t": "MINES_CHECK_ACTIVE",
  "p": {}
}
```

### 响应消息（有活跃游戏）

```json
{
  "i": "msg_013",
  "t": "MINES_CHECK_ACTIVE",
  "p": {
    "hasActiveGame": true,
    "roundId": "123456789012345678",
    "gameState": {
      "roundId": "123456789012345678",
      "status": "playing",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12],
      "safeTilesRevealed": 1
    }
  }
}
```

### 响应消息（无活跃游戏）

```json
{
  "i": "msg_013",
  "t": "MINES_CHECK_ACTIVE",
  "p": {
    "hasActiveGame": false
  }
}
```

## 6. MINES_RESUME_GAME - 恢复游戏

恢复中断的游戏，获取完整游戏状态。

### 请求消息

```json
{
  "i": "msg_014",
  "t": "MINES_RESUME_GAME",
  "p": {
    "roundId": "123456789012345678"  // 可选，不提供则恢复最新的活跃游戏
  }
}
```

### 响应消息

```json
{
  "i": "msg_014",
  "t": "MINES_RESUME_GAME",
  "p": {
    "success": true,
    "gameState": {
      "roundId": "123456789012345678",
      "status": "playing",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 8],
      "safeTilesRevealed": 2
    },
    "currentMultiplier": "1.09",
    "nextMultiplier": "1.14"
  }
}
```

## 7. 游戏规则

### 赔率计算

赔率基于揭开的安全格子数量动态计算：

```
基础赔率 = (总格子数 / (总格子数 - 地雷数)) ^ 揭开格子数
最终赔率 = 基础赔率 * 0.99 (RTP调整)
```

### 赔率示例（5x5网格，5个地雷）

| 揭开格子数 | 赔率 |
|------------|------|
| 1 | 1.04x |
| 2 | 1.09x |
| 3 | 1.14x |
| 5 | 1.26x |
| 10 | 1.65x |
| 15 | 2.38x |
| 20 | 4.50x |

### 游戏状态

- `playing`: 游戏进行中
- `finished`: 游戏已结束（包括提现、触雷等所有结束情况）

## 8. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `ACTIVE_SESSION_EXISTS` | 已有进行中的游戏 | 结束或恢复当前游戏 |
| `GAME_NOT_FOUND` | 游戏不存在 | 检查 roundId 是否正确 |
| `INVALID_TILE_INDEX` | 无效的格子索引 | 确保索引在有效范围内 |
| `TILE_ALREADY_REVEALED` | 格子已被揭开 | 选择其他格子 |
| `NO_TILES_REVEALED` | 未揭开任何格子 | 至少揭开一个格子后才能提现 |
| `INVALID_GRID_TYPE` | 无效的网格类型 | 使用 3x3、5x5 或 7x7 |
| `INVALID_MINES_COUNT` | 地雷数量无效 | 地雷数必须小于格子总数 |

## 9. 注意事项

1. **单游戏限制**：每个玩家同时只能有一个活跃的 Mines 游戏
2. **自动提现**：5分钟无操作且有已揭开格子时自动提现
3. **格子索引**：从0开始，左上角为0，按行递增
4. **最小提现条件**：至少揭开一个安全格子
5. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成
6. **断线恢复**：支持通过 MINES_RESUME_GAME 恢复游戏

## 10. 公平性验证 API

除了 WebSocket 接口外，系统还提供 RESTful API 用于独立验证游戏结果的公平性。

### 10.1 验证接口

**端点**: `POST /v1/fairness/mines/verify`

**认证**: 需要 JWT Token

**请求参数**:

```json
{
  "clientSeed": "player_seed_123",
  "serverSeed": "revealed_server_seed",
  "nonce": 1,
  "minesCount": 5,
  "gridType": "5x5"
}
```

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `clientSeed` | string | 是 | 客户端种子（游戏时提供的） |
| `serverSeed` | string | 是 | 服务器种子（游戏结束后揭示的） |
| `nonce` | number | 是 | Nonce 值（≥0） |
| `minesCount` | number | 是 | 地雷数量 |
| `gridType` | string | 是 | 网格类型："3x3"、"5x5"、"7x7" |

**响应结果**:

```json
{
  "minePositions": [3, 7, 15, 18, 22]
}
```

### 10.2 验证步骤

1. **获取游戏数据**：
   - 从游戏结果中获取 `provablyFair` 信息
   - 记录自己提供的 `clientSeed`

2. **调用验证接口**：
   ```bash
   curl -X POST https://dev.hicasino.xyz/v1/fairness/mines/verify \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_JWT_TOKEN" \
     -d '{
       "clientSeed": "your_client_seed",
       "serverSeed": "revealed_server_seed",
       "nonce": 1,
       "minesCount": 5,
       "gridType": "5x5"
     }'
   ```

3. **对比结果**：
   - 验证返回的地雷位置与游戏结果中的地雷位置是否一致
   - 如果一致，证明游戏结果公平

### 10.3 JavaScript 验证示例

```javascript
async function verifyMinesResult(gameResult, jwtToken) {
  const response = await fetch('https://dev.hicasino.xyz/v1/fairness/mines/verify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${jwtToken}`
    },
    body: JSON.stringify({
      clientSeed: gameResult.provablyFair.clientSeed,
      serverSeed: gameResult.provablyFair.serverSeed,
      nonce: gameResult.provablyFair.nonce,
      minesCount: gameResult.minesCount,
      gridType: gameResult.gridType
    })
  });

  const verification = await response.json();

  // 对比地雷位置
  const isValid = JSON.stringify(verification.minePositions.sort()) ===
                  JSON.stringify(gameResult.minePositions.sort());

  console.log('验证结果:', isValid ? '✅ 公平' : '❌ 不匹配');
  return isValid;
}
```

### 10.4 验证原理

验证接口使用与游戏相同的算法生成地雷位置：

```
对于每个地雷 i（i = 0 到 minesCount-1）：
  1. 组合种子：seedStr = "serverSeed:clientSeed:nonce:i"
  2. 计算哈希：hash = SHA256(seedStr)
  3. 转换为位置：position = hash 的前 8 位 % gridSize
  4. 处理碰撞：如果位置已占用，线性探测到下一个空位
```

这确保了：
- **确定性**：相同的种子组合总是产生相同的结果
- **不可预测性**：在服务器种子揭示前无法预测结果
- **可验证性**：任何人都可以独立验证结果

## 11. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Mines 游戏详细设计](./mines-detailed-design-zh.md)