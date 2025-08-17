# WebSocket API 接口文档

## 1. 概述

Invoker Server 提供基于 WebSocket 的实时游戏通信接口，用于处理游戏下注、状态同步、事件推送等功能。

### 1.1 连接信息

- **连接地址**: `wss://dev.hicasino.xyz/v1/ws`
- **协议**: WebSocket
- **消息格式**: JSON 或 Protocol Buffers
- **认证方式**: JWT Token（通过 URL 参数或 LOGIN 消息）
- **端口**: 8001

### 1.2 连接方式

#### 方式一：URL 参数认证（推荐）
```
wss://dev.hicasino.xyz/v1/ws?token={JWT_TOKEN}
```

#### 方式二：连接后发送 LOGIN 消息
1. 先建立 WebSocket 连接：`wss://dev.hicasino.xyz/v1/ws`
2. 发送 LOGIN 消息进行认证

### 1.3 消息格式

所有 WebSocket 消息使用统一的消息结构：

```json
{
  "i": "消息ID",      // 唯一消息ID，用于请求响应关联
  "t": "消息类型",    // 消息类型标识符
  "p": {             // 消息载荷，具体内容根据消息类型而定
    // payload data
  }
}
```

字段说明：
- `i` (id): 消息ID，客户端生成的唯一标识符，服务端响应时会返回相同的ID
- `t` (type): 消息类型，如 "LOGIN"、"PLACE_BET" 等
- `p` (payload): 消息载荷，包含具体的请求或响应数据

### 1.4 连接生命周期

1. **建立连接**: 客户端发起 WebSocket 连接
2. **认证**: 通过 JWT Token 认证（URL参数或LOGIN消息）
3. **初始化**: 服务端自动推送游戏配置和初始化完成事件
4. **消息交互**: 客户端发送请求，服务端返回响应或推送事件
5. **心跳保活**: 自动心跳机制保持连接活跃
6. **断开连接**: 客户端主动断开或超时断开

## 2. 通用接口

### 2.1 LOGIN - 登录认证

用于在连接建立后进行身份认证（如果未使用URL参数认证）。

**请求消息**：
```json
{
  "i": "msg_001",
  "t": "LOGIN",
  "p": {
    "token": "JWT_TOKEN_STRING"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_001",
  "t": "LOGIN",
  "p": {
    "success": true,
    "userId": "user_123",
    "gameId": "inhousegame:dice",
    "sessionId": "session_456"
  }
}
```

**错误响应**：
```json
{
  "i": "msg_001",
  "t": "LOGIN",
  "p": {
    "success": false,
    "error": {
      "code": "INVALID_TOKEN",
      "message": "Token验证失败"
    }
  }
}
```

### 2.2 GET_GAME_CONFIG - 获取游戏配置

获取游戏配置信息，包括投注限额、RTP等。

**请求消息**：
```json
{
  "i": "msg_002",
  "t": "GET_GAME_CONFIG",
  "p": {
    "gameId": "inhousegame:dice",  // 可选，不提供则返回所有游戏
    "allGames": false              // 是否返回所有游戏，默认false
  }
}
```

**响应消息**（Dice游戏示例）：
```json
{
  "i": "msg_002",
  "t": "GET_GAME_CONFIG",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:dice",
        "config": {
          "id": 1,
          "gameName": "Dice",
          "gameId": "inhousegame:dice",
          "category": "instant",
          "status": "active",
          "description": "Classic dice game",
          "minBet": 0.1,
          "maxBet": 10000,
          "rtp": 99,
          "defaultRTP": "99",
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 1,
              "minBet": 0.1,
              "maxBet": 10000,
              "maxProfit": 100000
            }
          ]
        }
      }
    ]
  }
}
```

### 2.3 GET_BALANCE - 获取余额

获取当前玩家余额。

**请求消息**：
```json
{
  "i": "msg_003",
  "t": "GET_BALANCE",
  "p": {
    "currency": "USD"  // 可选，不提供则使用默认货币
  }
}
```

**响应消息**：
```json
{
  "i": "msg_003",
  "t": "GET_BALANCE",
  "p": {
    "balance": "1000.50",
    "currency": "USD"
  }
}
```

### 2.4 GET_GAME_STATE - 获取游戏状态

获取当前游戏状态和服务器种子信息。

**请求消息**：
```json
{
  "i": "msg_004",
  "t": "GET_GAME_STATE",
  "p": {}
}
```

**响应消息**：
```json
{
  "i": "msg_004",
  "t": "GET_GAME_STATE",
  "p": {
    "balance": "1000.50",
    "gameState": "{\"activeGame\": false}",
    "serverSeedInfo": {
      "hashedServerSeed": "abc123...",
      "currentNonce": 42,
      "createdAt": 1704067200000
    }
  }
}
```

### 2.5 SUBSCRIBE - 订阅事件

订阅特定类型的事件推送。

**请求消息**：
```json
{
  "i": "msg_005",
  "t": "SUBSCRIBE",
  "p": {
    "eventTypes": ["BET_ACTIVITY", "BALANCE_UPDATE"],
    "filters": "{\"gameId\": \"inhousegame:dice\"}"  // 可选过滤条件
  }
}
```

**响应消息**：
```json
{
  "i": "msg_005",
  "t": "SUBSCRIBE",
  "p": {
    "subscriptionId": "sub_123",
    "eventTypes": ["BET_ACTIVITY", "BALANCE_UPDATE"],
    "success": true,
    "message": "订阅成功"
  }
}
```

### 2.6 UNSUBSCRIBE - 取消订阅

取消事件订阅。

**请求消息**：
```json
{
  "i": "msg_006",
  "t": "UNSUBSCRIBE",
  "p": {
    "eventTypes": ["BET_ACTIVITY"],  // 可选，不提供则取消所有订阅
    "subscriptionId": "sub_123"      // 可选
  }
}
```

**响应消息**：
```json
{
  "i": "msg_006",
  "t": "UNSUBSCRIBE",
  "p": {
    "success": true,
    "message": "取消订阅成功"
  }
}
```

### 2.7 GET_BET_ACTIVITIES - 获取投注活动历史

获取最近的投注活动记录。此接口不需要认证，允许未登录用户查看游戏热度。

**请求消息**：
```json
{
  "i": "msg_007",
  "t": "GET_BET_ACTIVITIES",
  "p": {
    "gameId": "inhousegame:dice",  // 必需，游戏ID
    "limit": 50                     // 可选，返回数量限制（1-100，默认50）
  }
}
```

**响应消息**：
```json
{
  "i": "msg_007",
  "t": "GET_BET_ACTIVITIES_RESPONSE",
  "p": {
    "activities": [
      {
        "betId": "bet_123",
        "playerId": "player_456",
        "playerName": "Alice***",
        "gameId": "inhousegame:dice",
        "betAmount": "100.00",
        "winAmount": "200.00",
        "currency": "USD",
        "multiplier": "2.00",
        "isWin": true,
        "timestamp": 1704067200000,
        "gameOutcome": {
          "diceOutcome": {
            "target": 50,
            "result": 45,
            "rollType": "UNDER"
          }
        }
      }
    ],
    "totalCount": 100,      // 该时间段内的总投注数（采样前）
    "sampledCount": 50       // 返回的实际数量（采样后）
  }
}
```

**字段说明**：
- `gameId`: 必需参数，指定要获取哪个游戏的投注活动
- `limit`: 可选参数，限制返回的活动数量
- `activities`: 投注活动列表，按时间倒序排列
- `totalCount`: 统计周期内的总投注数
- `sampledCount`: 实际返回的活动数量（可能因采样而少于总数）

**使用场景**：
1. 新用户连接后获取历史投注活动
2. 展示游戏热度和其他玩家的投注情况
3. 配合实时推送展示完整的投注流

**注意事项**：
- 此接口不需要用户认证
- 返回的玩家名称会进行脱敏处理
- 配合 BET_ACTIVITY_BATCH 事件订阅可实现完整的活动流展示

## 3. Dice 游戏接口

### 3.1 PLACE_BET - 下注

发起 Dice 游戏下注。

**请求消息**：
```json
{
  "i": "msg_007",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",
    "gameParams": {
      "diceParams": {
        "target": 50,      // 目标数字 (4-96)
        "condition": "OVER" // "OVER" 或 "UNDER"
      }
    },
    "clientSeed": "user_random_seed_123"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_007",
  "t": "PLACE_BET",
  "p": {
    "betId": "bet_789",
    "gameResult": {
      "gameId": "inhousegame:dice",
      "betAmount": "10.00",
      "winAmount": "20.00",
      "isWin": true,
      "diceOutcome": {
        "rolledNumber": 75.23,
        "target": 50,
        "condition": "OVER",
        "multiplier": "2.00"
      },
      "multiplier": "2.00",
      "timestamp": 1704067200000
    },
    "balance": "1010.50"
  }
}
```

### 3.2 GET_BET_HISTORY - 获取投注历史

获取玩家的投注历史记录。

**请求消息**：
```json
{
  "i": "msg_008",
  "t": "GET_BET_HISTORY",
  "p": {
    "limit": 10,
    "offset": 0
  }
}
```

**响应消息**：
```json
{
  "i": "msg_008",
  "t": "GET_BET_HISTORY",
  "p": {
    "history": [
      {
        "betId": "bet_789",
        "gameId": "inhousegame:dice",
        "betAmount": "10.00",
        "winAmount": "20.00",
        "isWin": true,
        "timestamp": 1704067200000
      }
    ],
    "total": 100
  }
}
```

## 4. Mines 游戏接口

### 4.1 PLACE_BET - 开始游戏

开始新的 Mines 游戏。

**请求消息**：
```json
{
  "i": "msg_009",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",
    "gameParams": {
      "minesParams": {
        "minesCount": 5,    // 地雷数量
        "gridType": "5x5"   // 网格类型
      }
    },
    "clientSeed": "user_random_seed_456"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_009",
  "t": "PLACE_BET",
  "p": {
    "betId": "bet_mines_123",
    "gameResult": {
      "gameId": "inhousegame:mines",
      "betAmount": "10.00",
      "winAmount": "0",
      "isWin": false,
      "minesOutcome": {
        "gameId": "game_mines_456",
        "status": "in_progress",
        "minesCount": 5,
        "gridType": "5x5",
        "revealedTiles": [],
        "safeTilesRevealed": 0,
        "currentMultiplier": "1.00",
        "nextMultiplier": "1.04"
      },
      "timestamp": 1704067200000
    },
    "balance": "990.00"
  }
}
```

### 4.2 MINES_REVEAL_TILE - 揭开格子

在 Mines 游戏中揭开一个格子。

**请求消息**：
```json
{
  "i": "msg_010",
  "t": "MINES_REVEAL_TILE",
  "p": {
    "gameId": "game_mines_456",
    "tileIndex": 12  // 格子索引 (0-24 for 5x5)
  }
}
```

**响应消息（安全格子）**：
```json
{
  "i": "msg_010",
  "t": "MINES_REVEAL_TILE",
  "p": {
    "isMine": false,
    "gameState": {
      "gameId": "game_mines_456",
      "status": "in_progress",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12],
      "safeTilesRevealed": 1,
      "createdAt": 1704067200000,
      "updatedAt": 1704067210000
    },
    "currentMultiplier": "1.04",
    "nextMultiplier": "1.09"
  }
}
```

**响应消息（触雷）**：
```json
{
  "i": "msg_010",
  "t": "MINES_REVEAL_TILE",
  "p": {
    "isMine": true,
    "gameState": {
      "gameId": "game_mines_456",
      "status": "lost",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 15],
      "safeTilesRevealed": 1,
      "createdAt": 1704067200000,
      "updatedAt": 1704067220000
    },
    "result": {
      "minePositions": [3, 7, 15, 18, 22],
      "safeTilesRevealed": 1,
      "finalMultiplier": "0",
      "payout": "0",
      "provablyFair": {
        "clientSeed": "user_random_seed_456",
        "serverSeed": "server_seed_revealed",
        "hashedServerSeed": "hash_abc123",
        "nonce": 1
      }
    }
  }
}
```

### 4.3 MINES_CASH_OUT - 结算提现

结算当前 Mines 游戏并提现。

**请求消息**：
```json
{
  "i": "msg_011",
  "t": "MINES_CASH_OUT",
  "p": {
    "gameId": "game_mines_456"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_011",
  "t": "MINES_CASH_OUT",
  "p": {
    "payout": "15.60",
    "gameState": {
      "gameId": "game_mines_456",
      "status": "cashed_out",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 8, 20],
      "safeTilesRevealed": 3,
      "createdAt": 1704067200000,
      "updatedAt": 1704067230000
    },
    "result": {
      "minePositions": [3, 7, 15, 18, 22],
      "safeTilesRevealed": 3,
      "finalMultiplier": "1.56",
      "payout": "15.60",
      "provablyFair": {
        "clientSeed": "user_random_seed_456",
        "serverSeed": "server_seed_revealed",
        "hashedServerSeed": "hash_abc123",
        "nonce": 1
      }
    },
    "balance": "1005.60"
  }
}
```

### 4.4 MINES_GET_STATE - 获取游戏状态

获取当前 Mines 游戏状态。

**请求消息**：
```json
{
  "i": "msg_012",
  "t": "MINES_GET_STATE",
  "p": {
    "gameId": "game_mines_456"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_012",
  "t": "MINES_GET_STATE",
  "p": {
    "gameState": {
      "gameId": "game_mines_456",
      "status": "in_progress",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 8],
      "safeTilesRevealed": 2,
      "createdAt": 1704067200000,
      "updatedAt": 1704067215000
    },
    "currentMultiplier": "1.09",
    "nextMultiplier": "1.14"
  }
}
```

### 4.5 MINES_CHECK_ACTIVE - 检查活跃游戏

检查是否有活跃的 Mines 游戏。

**请求消息**：
```json
{
  "i": "msg_013",
  "t": "MINES_CHECK_ACTIVE",
  "p": {}
}
```

**响应消息**：
```json
{
  "i": "msg_013",
  "t": "MINES_CHECK_ACTIVE",
  "p": {
    "hasActiveGame": true,
    "gameId": "game_mines_456",
    "gameState": {
      "gameId": "game_mines_456",
      "status": "in_progress",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12],
      "safeTilesRevealed": 1,
      "createdAt": 1704067200000,
      "updatedAt": 1704067210000
    }
  }
}
```

### 4.6 MINES_RESUME_GAME - 恢复游戏

恢复中断的 Mines 游戏。

**请求消息**：
```json
{
  "i": "msg_014",
  "t": "MINES_RESUME_GAME",
  "p": {
    "gameId": "game_mines_456"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_014",
  "t": "MINES_RESUME_GAME",
  "p": {
    "success": true,
    "gameState": {
      "gameId": "game_mines_456",
      "status": "in_progress",
      "betAmount": "10.00",
      "minesCount": 5,
      "gridType": "5x5",
      "revealedTiles": [12, 8],
      "safeTilesRevealed": 2,
      "createdAt": 1704067200000,
      "updatedAt": 1704067215000
    },
    "currentMultiplier": "1.09",
    "nextMultiplier": "1.14"
  }
}
```

### 4.7 MINES_ABANDON_GAME - 放弃游戏

放弃当前 Mines 游戏（不提现）。

**请求消息**：
```json
{
  "i": "msg_015",
  "t": "MINES_ABANDON_GAME",
  "p": {
    "gameId": "game_mines_456"
  }
}
```

**响应消息**：
```json
{
  "i": "msg_015",
  "t": "MINES_ABANDON_GAME",
  "p": {
    "success": true,
    "message": "游戏已放弃",
    "result": {
      "minePositions": [3, 7, 15, 18, 22],
      "safeTilesRevealed": 2,
      "finalMultiplier": "0",
      "payout": "0",
      "provablyFair": {
        "clientSeed": "user_random_seed_456",
        "serverSeed": "server_seed_revealed",
        "hashedServerSeed": "hash_abc123",
        "nonce": 1
      }
    }
  }
}
```

## 5. Keno 游戏接口

### 5.1 PLACE_BET - Keno下注

Keno是一个即时开奖的数字彩票游戏。玩家从1-40中选择1-10个数字，系统随机开出10个中奖号码。

**请求消息**：
```json
{
  "i": "msg_501",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",
    "clientSeed": "my_lucky_seed_123",
    "gameParams": {
      "kenoParams": {
        "selectedNumbers": [3, 7, 15, 22, 28]  // 选择1-10个数字（1-40范围内）
      }
    }
  }
}
```

**响应消息**：
```json
{
  "i": "msg_501",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "betId": "keno_bet_789",
    "gameResult": {
      "gameId": "inhousegame:keno",
      "betAmount": "10.00",
      "winAmount": "25.00",
      "isWin": true,
      "multiplier": "2.5",
      "timestamp": 1704067200000,
      "kenoOutcome": {
        "selectedNumbers": [3, 7, 15, 22, 28],
        "drawnNumbers": [3, 7, 9, 12, 15, 18, 22, 25, 31, 35],
        "matchedNumbers": [3, 7, 15, 22],
        "selectedCount": 5,
        "matchedCount": 4,
        "payout": "2.5"
      }
    },
    "balance": "1025.00"
  }
}
```

**游戏参数说明**：
- `selectedNumbers`: 玩家选择的数字数组
  - 数字范围：1-40
  - 数量限制：1-10个
  - 不能重复

**游戏结果说明**：
- `drawnNumbers`: 系统开出的10个中奖号码
- `matchedNumbers`: 玩家选中的号码
- `matchedCount`: 命中数量
- `payout`: 赔率倍数

**赔付表示例**（选择5个数字）：
| 命中数量 | 赔率 |
|---------|------|
| 0 | 0x |
| 1 | 0x |
| 2 | 0x |
| 3 | 1x |
| 4 | 2.5x |
| 5 | 10x |

**特点**：
1. **即时游戏**：一次下注立即开奖，无需会话管理
2. **灵活选择**：可选择1-10个数字，不同选择数量有不同的赔付表
3. **可证明公平**：使用标准的provably fair算法生成开奖结果

**错误码**：
- `INVALID_NUMBER_COUNT`: 选择的数字数量不在1-10范围内
- `INVALID_NUMBER_RANGE`: 选择的数字不在1-40范围内
- `DUPLICATE_NUMBERS`: 选择的数字有重复

## 6. 事件推送

服务端会主动推送以下事件到客户端。

### 6.1 INITIALIZATION_COMPLETE - 初始化完成

连接建立并认证成功后，服务端推送初始化完成事件。

**推送消息**：
```json
{
  "i": "server_msg_001",
  "t": "INITIALIZATION_COMPLETE",
  "p": {
    "message": "WebSocket connection initialized successfully",
    "gameId": "inhousegame:dice",
    "timestamp": 1704067200000
  }
}
```

### 6.2 BALANCE_UPDATE - 余额更新

当玩家余额发生变化时推送。

**推送消息**：
```json
{
  "i": "server_msg_002",
  "t": "BALANCE_UPDATE",
  "p": {
    "playerId": "player_123",
    "currency": "USD",
    "balance": "1050.00",
    "changeAmount": "50.00",
    "reason": "WIN",
    "gameId": "inhousegame:dice",
    "timestamp": 1704067210000
  }
}
```

### 6.3 BET_ACTIVITY - 投注活动广播

实时广播其他玩家的投注活动（需要先订阅）。

**推送消息**：
```json
{
  "i": "server_msg_003",
  "t": "BET_ACTIVITY",
  "p": {
    "playerId": "player_456",
    "playerName": "Player456",
    "gameId": "inhousegame:dice",
    "gameName": "Dice",
    "betAmount": "100.00",
    "winAmount": "200.00",
    "multiplier": "2.00",
    "isWin": true,
    "currency": "USD",
    "timestamp": 1704067220000
  }
}
```

## 7. 错误处理

### 7.1 错误响应格式

当请求处理失败时，服务端返回错误响应。

```json
{
  "i": "msg_id",
  "t": "ERROR",
  "p": {
    "code": "ERROR_CODE",
    "message": "错误描述",
    "details": "{}",  // 额外错误信息（JSON字符串）
    "requestId": "original_msg_id"
  }
}
```

### 7.2 常见错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `UNAUTHORIZED` | 未认证或认证失败 | 重新发送LOGIN消息或检查JWT Token |
| `INVALID_TOKEN` | Token无效或过期 | 获取新的JWT Token |
| `INVALID_PARAMS` | 请求参数无效 | 检查请求参数格式和值 |
| `INSUFFICIENT_BALANCE` | 余额不足 | 提示用户充值 |
| `GAME_NOT_FOUND` | 游戏不存在 | 检查gameId是否正确 |
| `GAME_ALREADY_ACTIVE` | 已有进行中的游戏 | 结束或恢复当前游戏 |
| `INVALID_GAME_STATE` | 游戏状态无效 | 检查游戏当前状态 |
| `BET_LIMIT_EXCEEDED` | 超出投注限额 | 调整投注金额 |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 提供8-256字符的种子 |
| `INTERNAL_ERROR` | 服务器内部错误 | 稍后重试或联系支持 |

## 8. 注意事项

### 8.1 连接管理
- WebSocket连接会自动维护心跳，无需手动发送ping/pong
- 连接断开后需要重新认证
- 建议实现断线重连机制

### 8.2 消息ID管理
- 客户端应为每个请求生成唯一的消息ID
- 通过消息ID关联请求和响应
- 建议使用UUID或时间戳+随机数

### 8.3 数值精度
- 所有金额相关字段使用字符串类型，避免浮点数精度问题
- 示例："10.50" 而不是 10.5

### 8.4 时间戳格式
- 所有时间戳使用Unix毫秒时间戳（int64）
- 示例：1704067200000

### 8.5 游戏ID格式
- 游戏ID格式为："inhousegame:游戏类型"
- 示例："inhousegame:dice"、"inhousegame:mines"、"inhousegame:keno"

### 8.6 客户端种子
- 必须提供8-256个字符的客户端种子
- 用于可证明公平机制
- 每次游戏建议使用不同的种子

## 9. 实现状态

| 功能模块 | 状态 | 说明 |
|----------|------|------|
| WebSocket连接 | ✅ 已实现 | 支持JWT认证 |
| 通用接口 | ✅ 已实现 | 所有通用接口已完成 |
| Dice游戏 | ✅ 已实现 | 完整游戏流程已实现 |
| Mines游戏 | ✅ 已实现 | 完整游戏流程已实现 |
| Keno游戏 | ✅ 已实现 | 即时开奖，无需会话管理 |
| 投注活动历史 | ✅ 已实现 | GET_BET_ACTIVITIES接口 |
| 事件推送 | ✅ 已实现 | 支持余额更新和投注广播 |
| 可证明公平 | ✅ 已实现 | 完整的provably fair机制 |
| Blackjack游戏 | 🚧 开发中 | 基础功能已实现 |