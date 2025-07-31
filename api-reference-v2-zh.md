# API 参考 v2 - Invoker Server

## 概述

Invoker Server v2 提供了统一、简洁的 API 结构，作为游戏提供商为各类客户端（玩家、赌场平台、聚合器）提供游戏服务。

### 核心理念
- **Invoker 是游戏提供商**：专注于提供高质量的游戏体验
- **统一接口**：所有客户端使用相同的游戏 API
- **清晰分层**：游戏业务与聚合器管理分离

## API 结构

```
/v1/
├── game/          # 游戏相关接口
│   ├── games      # 游戏列表和配置
│   ├── sessions   # 会话管理
│   ├── bets       # 投注操作
│   └── history    # 历史记录
├── aggregator/    # 聚合器管理
│   └── ...        # CRUD 操作
└── ws            # WebSocket 连接
```

## 1. Game API

### 1.1 游戏信息

#### 获取游戏列表
**GET** `/v1/games`

获取所有可用游戏的列表。

**请求参数**：
- `category` (可选): 游戏类别筛选
- `status` (可选): 游戏状态筛选

**响应示例**：
```json
{
  "games": [
    {
      "id": "inhousegame:dice",
      "name": "Dice",
      "description": "Classic dice game with adjustable win chance",
      "category": "instant",
      "status": "active",
      "thumbnail_url": "/games/dice/thumb.png",
      "features": ["provably_fair", "instant_play"],
      "limits": {
        "currency_limits": [
          {
            "currency": "USD",
            "currency_type": "fiat",
            "min_bet": "0.10",
            "max_bet": "10000.00",
            "default_bet": "1.00"
          }
        ]
      },
      "rtp": 98.0
    }
  ]
}
```

#### 获取游戏配置
**GET** `/v1/games/{game_id}/config`

获取特定游戏的详细配置。

**响应示例**：
```json
{
  "game": {
    "id": "inhousegame:dice",
    "name": "Dice",
    // ... 基本信息
  },
  "config": {
    "house_edge": 2.0,
    "min_target": 0.01,
    "max_target": 98.99,
    "decimal_places": 2
  }
}
```

### 1.2 会话管理

#### 创建游戏会话
**POST** `/v1/sessions`

为玩家创建新的游戏会话。

**请求体**：
```json
{
  "player_id": "player_123",
  "game_id": "inhousegame:dice",
  "currency": "USD",
  "aggregator_id": "casino_xyz",
  "params": {
    "language": "zh",
    "return_url": "https://casino.com/lobby"
  }
}
```

**响应**：
```json
{
  "session_id": "sess_abc123",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_at": "2024-01-20T18:00:00Z",
  "expires_in": 7200,
  "game_url": "https://games.invoker.com/play?token=..."
}
```

#### 获取会话信息
**GET** `/v1/sessions/{session_id}`

**响应**：
```json
{
  "session": {
    "session_id": "sess_abc123",
    "player_id": "player_123",
    "game_id": "inhousegame:dice",
    "currency": "USD",
    "status": "active",
    "created_at": "2024-01-20T16:00:00Z"
  },
  "in_game": true,
  "last_activity": "2024-01-20T16:30:00Z"
}
```

### 1.3 投注操作

#### 下注
**POST** `/v1/bets`

执行游戏投注。

**请求体（Dice 示例）**：
```json
{
  "session_id": "sess_abc123",
  "game_id": "inhousegame:dice",
  "amount": {
    "amount": "10.00000000",
    "currency": "USD"
  },
  "client_seed": "my-lucky-seed",
  "game_params": {
    "dice": {
      "target": 50.5,
      "is_roll_over": true
    }
  }
}
```

**响应**：
```json
{
  "result": {
    "bet_id": "bet_789",
    "session_id": "sess_abc123",
    "bet_amount": {
      "amount": "10.00000000",
      "currency": "USD"
    },
    "win_amount": {
      "amount": "20.00000000",
      "currency": "USD"
    },
    "is_win": true,
    "multiplier": "2.00000000",
    "created_at": "2024-01-20T16:35:00Z"
  },
  "provably_fair": {
    "client_seed": "my-lucky-seed",
    "server_seed": "server-seed-revealed",
    "hashed_server_seed": "sha256-hash",
    "nonce": 42
  },
  "game_result": {
    "dice": {
      "roll": 75.23,
      "target": 50.5,
      "is_roll_over": true
    }
  }
}
```

#### 游戏动作（多步游戏）
**POST** `/v1/games/{game_id}/action`

用于需要多步操作的游戏（如 Mines、Blackjack）。

**请求体（Mines 揭示瓦片）**：
```json
{
  "game_id": "mines_game_123",
  "session_id": "sess_abc123",
  "action": {
    "mines": {
      "reveal": {
        "tile_index": 12
      }
    }
  }
}
```

**响应**：
```json
{
  "action_completed": true,
  "game_status": "in_progress",
  "result": {
    "mines": {
      "is_mine": false,
      "game_state": {
        "game_id": "mines_game_123",
        "status": "in_progress",
        "revealed_tiles": [3, 7, 12],
        "safe_tiles_revealed": 3,
        "current_multiplier": "1.13",
        "next_multiplier": "1.19"
      }
    }
  }
}
```

### 1.4 历史记录

#### 获取玩家投注历史
**POST** `/v1/bets/page`

**请求参数**：
- `player_id`: 玩家 ID (必填)
- `game_id` (可选): 游戏筛选
- `currency` (可选): 货币筛选
- `start_time` (可选): 开始时间
- `end_time` (可选): 结束时间
- `pagination.page_size`: 每页数量
- `pagination.page_token`: 分页标记

**响应**：
```json
{
  "bets": [
    {
      "bet": {
        "bet_id": "bet_789",
        "bet_amount": {"amount": "10.00", "currency": "USD"},
        "win_amount": {"amount": "20.00", "currency": "USD"},
        "is_win": true,
        "created_at": "2024-01-20T16:35:00Z"
      },
      "game_id": "inhousegame:dice",
      "game_name": "Dice",
      "game_data": {
        "roll": 75.23,
        "target": 50.5
      }
    }
  ],
  "pagination": {
    "next_page_token": "eyJvZmZzZXQ..."
  },
  "summary": {
    "total_bets": 150,
    "total_wagered": {"amount": "1500.00", "currency": "USD"},
    "total_won": {"amount": "1600.00", "currency": "USD"},
    "net_profit": {"amount": "100.00", "currency": "USD"},
    "win_rate": 48.5
  }
}
```

## 2. WebSocket API

### 连接流程

1. 建立连接：`wss://api.invoker.com/v1/ws`
2. 发送登录消息进行认证
3. 接收游戏配置和实时事件

### 消息格式

所有 WebSocket 消息使用统一的封装格式：

```json
{
  "id": "msg_123",
  "type": "MESSAGE_TYPE",
  "timestamp": "2024-01-20T16:35:00Z",
  "payload": { ... }
}
```

### 主要消息类型

#### 登录认证
```json
// 请求
{
  "id": "msg_login_1",
  "type": "LOGIN",
  "payload": {
    "token": "eyJhbGciOiJIUzI1NiIs..."
  }
}

// 响应
{
  "id": "msg_login_resp_1",
  "type": "LOGIN_RESPONSE",
  "payload": {
    "success": true,
    "user_id": "12345",
    "session_id": "sess_abc123",
    "game_id": "inhousegame:dice"
  }
}
```

#### 获取余额
```json
// 请求
{
  "id": "msg_balance_1",
  "type": "GET_BALANCE",
  "payload": {
    "currency": "USD"  // 可选
  }
}

// 响应
{
  "id": "msg_balance_resp_1",
  "type": "GET_BALANCE_RESPONSE",
  "payload": {
    "balance": "1000.00000000",
    "currency": "USD"
  }
}
```

#### 下注（通过 WebSocket）
```json
// 请求
{
  "id": "msg_bet_1",
  "type": "PLACE_BET",
  "payload": {
    // 与 HTTP API 的 PlaceBetRequest 相同
  }
}

// 响应
{
  "id": "msg_bet_resp_1",
  "type": "PLACE_BET_RESPONSE",
  "payload": {
    // 与 HTTP API 的 PlaceBetResponse 相同
  }
}
```

#### 实时事件推送
```json
{
  "id": "msg_event_1",
  "type": "GAME_EVENT",
  "payload": {
    "event_id": "evt_123",
    "event_type": "BET_ACTIVITY_BATCH",
    "timestamp": "2024-01-20T16:35:00Z",
    "data": {
      "bet_activity_batch": {
        "activities": [
          {
            "masked_player_id": "pla***567",
            "bet_amount": "100.00",
            "currency": "USD",
            "is_win": true,
            "win_amount": "500.00",
            "multiplier": "5.00",
            "game_id": "inhousegame:dice"
          }
        ],
        "total_count": 15,
        "sampled_count": 1
      }
    }
  }
}
```

## 3. Aggregator API

聚合器 API 用于管理接入的赌场平台。

### 主要功能
- 创建和管理聚合器账户
- API 密钥管理
- Webhook 配置
- IP 白名单设置

详细文档请参考现有的 Aggregator API 文档。

## 4. 认证方式

### HTTP API
- **聚合器认证**：使用 HMAC-SHA256 签名
- **会话认证**：使用 JWT Bearer Token

### WebSocket
- 使用 JWT Token 通过 LOGIN 消息认证

## 5. 错误处理

### 错误响应格式
```json
{
  "code": "INSUFFICIENT_BALANCE",
  "message": "余额不足",
  "metadata": {
    "required_amount": "100.00",
    "current_balance": "50.00"
  }
}
```

### 错误码范围
- 1000-9999：通用错误
- 10000-19999：游戏相关错误
- 20000-29999：聚合器相关错误

## 6. 最佳实践

1. **金额处理**：所有金额使用字符串格式，保留 8 位小数
2. **幂等性**：使用唯一的请求 ID 确保操作幂等
3. **错误重试**：实现指数退避的重试机制
4. **WebSocket 重连**：断线后自动重连并重新认证

## 7. 迁移指南

从旧 API 迁移到新 API，请参考 [API 迁移指南](../api/migration_guide.md)。