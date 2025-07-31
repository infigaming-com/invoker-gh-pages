# 统一 API 参考 - Invoker Server v2

## 概述

Invoker Server v2 采用了全新的统一 API 设计，将原有的分散接口整合为清晰的服务边界：

- **Game API** (`/api/game/v1/`) - 统一的游戏服务接口
- **Aggregator API** (`/api/aggregator/v1/`) - 聚合器管理接口
- **Provider API** (`/api/provider/v1/`) - 内部提供商接口（计划移除）

## Game API - 统一游戏接口

### 服务端点

- **HTTP/gRPC**: `http://dev.hicasino.xyz:8000/v1/game`
- **WebSocket**: `ws://dev.hicasino.xyz:8001/v1/game/ws`

### 1. 游戏管理服务 (GameService)

#### 获取游戏列表
```protobuf
rpc GetGames(GetGamesRequest) returns (GetGamesResponse) {
  option (google.api.http) = {
    get: "/v1/games"
  };
}
```

**请求参数**:
- `category` (string, optional) - 游戏类别筛选
- `status` (string, optional) - 游戏状态筛选 (active/inactive)

**响应示例**:
```json
{
  "games": [
    {
      "id": "dice",
      "name": "骰子游戏",
      "description": "经典的大小猜测游戏",
      "category": "dice",
      "status": "active",
      "thumbnail_url": "/images/dice.png",
      "features": ["provably_fair", "instant_play"],
      "rtp": 98.0,
      "limits": {
        "currency_limits": [
          {
            "currency": "USD",
            "currency_type": "fiat",
            "min_bet": "0.10000000",
            "max_bet": "1000.00000000",
            "default_bet": "1.00000000",
            "max_profit": "10000.00000000"
          }
        ]
      }
    }
  ]
}
```

#### 获取游戏配置
```protobuf
rpc GetGameConfig(GetGameConfigRequest) returns (GetGameConfigResponse) {
  option (google.api.http) = {
    get: "/v1/games/{game_id}/config"
  };
}
```

### 2. 会话管理服务 (SessionService)

#### 创建游戏会话
```protobuf
rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse) {
  option (google.api.http) = {
    post: "/v1/sessions"
    body: "*"
  };
}
```

**请求参数**:
```json
{
  "player_id": "player123",
  "game_id": "dice",
  "currency": "USD",
  "aggregator_id": "io_aggregator",
  "params": {
    "language": "zh-CN",
    "return_url": "https://casino.com/lobby",
    "metadata": {
      "campaign": "welcome_bonus"
    }
  }
}
```

**响应示例**:
```json
{
  "session_id": "session_player123_1234567890",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_at": "2024-01-01T12:00:00Z",
  "expires_in": 7200,
  "game_url": "/play?token=eyJhbGciOiJIUzI1NiIs..."
}
```

### 3. 投注服务 (BetService)

#### 下注
```protobuf
rpc PlaceBet(PlaceBetRequest) returns (PlaceBetResponse) {
  option (google.api.http) = {
    post: "/v1/bets"
    body: "*"
  };
}
```

**请求参数（Dice 示例）**:
```json
{
  "session_id": "session_player123_1234567890",
  "amount": {
    "amount": "10.00000000",
    "currency": "USD"
  },
  "client_seed": "player_random_seed",
  "dice": {
    "target": 50.0,
    "is_roll_over": true
  }
}
```

**响应示例**:
```json
{
  "result": {
    "bet_id": "bet_123456",
    "session_id": "session_player123_1234567890",
    "bet_amount": {
      "amount": "10.00000000",
      "currency": "USD"
    },
    "win_amount": {
      "amount": "19.60000000",
      "currency": "USD"
    },
    "is_win": true,
    "multiplier": "1.96000000",
    "created_at": "2024-01-01T10:00:00Z"
  },
  "provably_fair": {
    "client_seed": "player_random_seed",
    "server_seed": "server_seed_revealed",
    "hashed_server_seed": "hashed_server_seed",
    "nonce": 1
  },
  "dice": {
    "roll": 75.23,
    "target": 50.0,
    "is_roll_over": true
  }
}
```

### 4. 历史记录服务 (HistoryService) ⚠️ 待实现

#### 获取投注历史
```protobuf
rpc GetBetHistory(GetBetHistoryRequest) returns (GetBetHistoryResponse) {
  option (google.api.http) = {
    get: "/v1/players/{player_id}/bets"
  };
}
```

### 5. WebSocket 服务

WebSocket 连接用于实时游戏通信和事件推送。

**连接地址**: `ws://dev.hicasino.xyz:8001/v1/game/ws?token={jwt_token}`

**消息格式**:
```json
{
  "type": "bet",
  "payload": {
    // 具体的游戏操作数据
  }
}
```

## Aggregator API - 聚合器管理接口

### 服务端点

- **HTTP/gRPC**: `http://dev.hicasino.xyz:8000/v1/aggregator`

### 聚合器管理

#### 创建聚合器
```protobuf
rpc CreateAggregator(CreateAggregatorRequest) returns (CreateAggregatorResponse) {
  option (google.api.http) = {
    post: "/v1/aggregators"
    body: "*"
  };
}
```

#### 列出聚合器
```protobuf
rpc ListAggregators(ListAggregatorsRequest) returns (ListAggregatorsResponse) {
  option (google.api.http) = {
    get: "/v1/aggregators"
  };
}
```

## 错误处理

### 错误代码范围

- **1000-1999**: 通用错误
- **2000-2999**: 认证和授权错误
- **10000-19999**: 游戏相关错误
- **20000-29999**: 聚合器相关错误
- **30000-39999**: Provider 相关错误

### 错误响应格式

```json
{
  "code": 10001,
  "reason": "INVALID_BET_AMOUNT",
  "message": "投注金额必须大于最小限额",
  "metadata": {
    "min_bet": "0.10000000",
    "max_bet": "1000.00000000"
  }
}
```

## 认证机制

### JWT Token

所有游戏相关的 API 调用都需要携带有效的 JWT token。

**Token 格式**:
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Token 内容**:
```json
{
  "session_id": "session_player123_1234567890",
  "user_id": 999999,
  "player_id": "player123",
  "aggregator_id": "io_aggregator",
  "game_id": "dice",
  "operator_id": "casino_operator",
  "currency": "USD",
  "exp": 1234567890
}
```

### HMAC 签名（仅 Provider API）

Provider API 使用 HMAC-SHA256 签名验证，详见 Provider API 文档。

## 迁移指南

### 从旧 API 迁移到新 API

1. **游戏前端调用**:
   - 旧: `/v1/frontend/games/dice/bet`
   - 新: `/v1/bets` (使用统一的 PlaceBet 接口)

2. **会话管理**:
   - 旧: 分散在各个游戏服务中
   - 新: 统一的 SessionService

3. **游戏配置**:
   - 旧: 每个游戏独立的配置接口
   - 新: 统一的 GetGameConfig 接口

### 主要变化

1. **统一的数据模型**: 所有游戏使用相同的基础数据结构
2. **标准化的错误处理**: 统一的错误代码和响应格式
3. **简化的认证流程**: JWT token 统一管理
4. **更好的扩展性**: 新增游戏只需实现相应的参数结构

## 开发计划

### 已完成 ✅
- Game API 基础结构
- Session 管理服务
- Dice 游戏适配
- 基础 Bet 服务

### 进行中 🚧
- Mines 游戏适配（暂停）
- Blackjack 游戏适配（暂停）

### 待开发 ⏳
- History 服务（需要 BetRepo 实现）
- 统一的游戏状态管理
- 实时推送优化
- 完整的 API 网关功能