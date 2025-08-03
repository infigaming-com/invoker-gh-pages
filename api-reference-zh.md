# API 参考 - Invoker Server

## 重要说明

本文档包含 Invoker Server 的所有 API 接口说明，包括当前已实现的接口和未来规划的接口。

### API 实现状态

#### ✅ 已实现的 API
- **Provider API** (`/api/provider/v1/`) - 游戏聚合器接口
- **WebSocket API** (`wss://dev.hicasino.xyz/v1/ws`) - 实时游戏通信
- **Aggregator API** (`/api/aggregator/v1/`) - 聚合器管理
- **游戏前端 API** - Dice、Mines、Blackjack 游戏端点

#### ⚠️ 未来规划（待实现）
- **统一 Game API** (`/v1/game/*`) - 计划中的统一游戏接口
  - 目标：将分散的游戏接口整合为统一的服务
  - 当前状态：设计阶段，尚未实现
  - 预计收益：简化接口调用，提高开发效率

### 当前使用指南
请使用已实现的 API 接口进行集成。统一 Game API 仍在规划中，实现时间待定。

## API 认证方式总览

不同的 API 使用不同的认证机制：

1. **Provider API** (`/api/provider/v1/*`) - HMAC-SHA256 签名认证
2. **Game API** (`/api/game/v1/*`) - JWT Bearer token 认证 
3. **WebSocket API** (`/v1/ws`) - 通过 LOGIN 消息使用 JWT token
4. **Aggregator API** (`/api/aggregator/v1/*`) - 主密钥认证（管理接口）

## 目录

### 已实现接口
1. [Provider API](#provider-api) ✅ 生产可用
2. [WebSocket API](#websocket-api) ✅ 生产可用
3. [游戏前端 API](#游戏前端-api) ✅ 生产可用
4. [Aggregator API](#aggregator-api) ✅ 生产可用
5. [错误代码](#错误代码)
6. [金额格式说明](#金额格式说明)

### 未来规划
7. [统一 Game API（设计阶段）](#统一-game-api设计阶段) ⚠️ 待实现
8. [集成 API（未实现）](#集成-api) ⚠️ 无实现

## Provider API

### 概述
Provider API 是为游戏聚合器（GA）设计的标准化接口，允许赌场平台通过游戏聚合器访问我们的游戏。

**基础信息**:
- **端点**: `https://dev.hicasino.xyz/v1/provider`
- **端口**: 8000（与主 HTTP 服务共用）
- **协议**: HTTP/HTTPS
- **认证**: HMAC-SHA256 签名验证 ✅ *已实现*

**实现状态**:
- ✅ 所有接口已实现并可用
- ✅ HMAC-SHA256 认证中间件已实现
- ✅ JWT token 生成和管理已实现
- ✅ 完整的错误处理和日志记录

**架构更新**:
- ✅ 余额管理功能已移除，由聚合器（io）负责
- ✅ 游戏提供商专注于游戏逻辑，不处理金钱交易
- ✅ 所有余额检查、扣款、加款由聚合器在调用前后处理

### 认证

所有请求必须包含以下头部：
- `X-API-KEY`: API密钥
- `X-SIGNATURE`: 请求签名
- `Content-Type`: application/json

### 接口列表

#### 1. 创建会话
**POST** `/api/provider/v1/sessions`

创建新的游戏会话，返回游戏URL供玩家访问。

**请求参数说明**:
- `player_id` (必需): 玩家唯一标识符
- `game_id` (必需): 游戏ID，必须是系统中存在且处于活跃状态的游戏
  - 格式："inhousegame:游戏名"，如 "inhousegame:dice"、"inhousegame:mines"、"inhousegame:blackjack"
  - 系统会验证游戏是否存在且状态为 "active"
- `currency` (必需): 货币代码（如 USD、EUR、BTC）
- `operator_id` (必需): 运营商ID
- `session_params` (可选): 会话参数
  - `language`: 语言代码
  - `return_url`: 返回URL

**请求体示例**:
```json
{
  "player_id": "player_123",
  "game_id": "inhousegame:dice",
  "currency": "USD",
  "operator_id": "ga_001",
  "session_params": {
    "language": "en",
    "return_url": "https://casino.com/games"
  }
}
```

**成功响应**:
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_at": "2024-01-20T17:30:00Z",
    "expires_in": 7200
  }
}
```

**错误响应示例**:
```json
{
  "success": false,
  "error": {
    "code": "MISSING_PARAMETER",
    "message": "game_id is required"
  }
}
```

```json
{
  "success": false,
  "error": {
    "code": "GAME_NOT_FOUND",
    "message": "Invalid or inactive game_id"
  }
}
```

**Token 说明**:
- 返回的 token 是 JWT 格式，根据请求参数生成
- JWT claims 包含：
  - `session_id`: 生成的会话ID
  - `user_id`: 内部用户ID（int64格式）
  - `aggregator_id`: 聚合器标识（如 "io", "ga1"）
  - `game_id`: 请求中的游戏ID
  - `operator_id`: 请求中的运营商ID
  - `currency`: 请求中的货币代码
- Token 有效期 2 小时，可用于 WebSocket 连接认证
- WebSocket 连接需通过 LOGIN 消息进行认证

#### 2. 获取会话信息
**GET** `/api/provider/v1/sessions/{session_id}`

查询会话状态和游戏进度。

**响应**:
```json
{
  "success": true,
  "data": {
    "session_id": "sess_abc123",
    "player_id": "player_123",
    "game_id": "dice",
    "status": "active",
    "created_at": "2024-01-20T15:30:00Z",
    "last_activity": "2024-01-20T15:35:00Z",
    "game_state": {
      "current_round": "round_123",
      "in_progress": false
    }
  }
}
```

#### 3. 下注游戏
**POST** `/api/provider/v1/play`

处理玩家下注请求。

**请求体**:
```json
{
  "session_id": "sess_abc123",
  "round_id": "round_123",
  "bet_amount": "10.00000000",
  "game_params": {
    "target": 50.00,
    "direction": "over"
  },
  "client_seed": "client_seed_xyz"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "round_id": "round_123",
    "transaction_id": "tx_456",
    "result": {
      "outcome": 75.23,
      "win": true,
      "win_amount": "20.00000000",
      "multiplier": "2.00000000"
    },
    "provably_fair": {
      "server_seed": "server_seed_abc",
      "client_seed": "client_seed_xyz",
      "nonce": 1
    }
  }
}
```

> ✅ **注意**：余额信息不再由游戏提供商返回，由聚合器管理。

#### 4. 查询余额（已移除）
> ✅ **架构更新**：余额管理功能已移除，现在由聚合器（io）负责管理玩家余额。

#### 5. 交易回滚（已更新）
**POST** `/api/provider/v1/rollback`

> ✅ **架构更新**：回滚功能由聚合器（io）处理。此端点现在返回不支持的提示。

**响应**:
```json
{
  "success": false,
  "error": {
    "code": "NOT_SUPPORTED",
    "message": "Rollback is handled by aggregator, not game provider"
  }
}
```

#### 6. 获取游戏列表
**GET** `/api/provider/v1/games?status=active`

查询可用游戏列表。

**响应**:
```json
{
  "success": true,
  "data": {
    "games": [
      {
        "game_id": "inhousegame:dice",
        "name": "Dice Game",
        "category": "instant",
        "status": "active",
        "min_bet": 0.10,
        "max_bet": 1000.00,
        "rtp": 98.0,
        "features": ["provably_fair", "instant_play", "auto_play"]
      }
    ]
  }
}
```


### Provider API 错误码

| 错误码 | 含义 | HTTP状态 |
|--------|------|-----------|
| MISSING_SIGNATURE | 缺少签名 | 401 |
| INVALID_SIGNATURE | 签名无效 | 401 |
| EXPIRED_REQUEST | 请求过期 | 408 |
| MISSING_PARAMETER | 缺少必需参数 | 400 |
| INVALID_PARAMETER | 参数值无效 | 400 |
| SESSION_NOT_FOUND | 会话不存在 | 404 |
| SESSION_EXPIRED | 会话已过期 | 403 |
| GAME_NOT_FOUND | 游戏不存在 | 404 |
| INSUFFICIENT_BALANCE | 余额不足 | 402 |
| TRANSACTION_NOT_FOUND | 交易不存在 | 404 |

## WebSocket API

### 消息格式更新说明

从 2025 年 1 月起，部分消息类型已从使用 `d` 字段（字符串）迁移到使用 `p` 字段（结构化对象）：

| 消息类型 | 旧格式 | 新格式 | 说明 |
|----------|--------|--------|------|
| INITIALIZATION_COMPLETE | `d` 字段（JSON字符串） | `p` 字段（结构化对象） | 初始化完成通知 |
| GAME_CONFIG | `d` 字段（JSON字符串） | 保持不变 | 其他游戏配置（向后兼容） |
| gameOutcome | JSON字符串 | 结构化对象（oneof） | 游戏结果数据 |
| BALANCE_UPDATE | - | `p` 字段（结构化对象） | 余额更新通知 ✅ *新增* |

**游戏结果结构化类型**：
- `gameOutcome` 字段现在使用 `oneof` 类型，根据不同游戏返回对应的结构化数据
- 支持的类型：`diceOutcome`、`minesOutcome`、`blackjackOutcome`
- 每种游戏结果都有明确的字段定义，提供更好的类型安全性

**迁移说明**：
- 新格式提供类型安全和更好的开发体验
- 所有新开发的消息类型都将使用 `p` 字段
- 现有的其他消息类型保持向后兼容

### 1. 端点信息

**WebSocket端点**: `wss://dev.hicasino.xyz/v1/ws`

### 2. 认证

**认证方式**: LOGIN消息认证

WebSocket连接建立后，必须通过发送LOGIN消息进行身份认证：

**连接URL**: `wss://dev.hicasino.xyz/v1/ws`

**LOGIN请求消息**:
```json
{
  "i": "msg_login_123",
  "t": "LOGIN",
  "p": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**LOGIN响应消息（成功）**:
```json
{
  "i": "msg_login_resp_123",
  "t": "LOGIN_RESPONSE",
  "p": {
    "success": true,
    "user_id": "12345",
    "game_id": "inhousegame:dice",
    "session_id": "sess_abc123"
  }
}
```

**LOGIN响应消息（失败）**:
```json
{
  "i": "msg_login_resp_123",
  "t": "LOGIN_RESPONSE",
  "p": {
    "success": false,
    "error": {
      "code": "INVALID_TOKEN",
      "message": "Token is invalid or expired"
    }
  }
}
```

**优势**:
- 更安全：token不会出现在服务器日志中
- 更灵活：支持先连接后认证
- 可扩展：支持未认证用户接收公共消息

**令牌获取**: 必须通过Provider API的[CreateSession](#1-创建会话)接口获取

### 3. 连接初始化流程

**连接建立流程**:
1. 建立WebSocket连接
2. 客户端可立即发送 `GET_GAME_CONFIG` 请求获取游戏配置（无需登录）
3. 客户端必须在30秒内发送 `LOGIN` 消息进行认证
4. 服务器验证token并响应 `LOGIN_RESPONSE`
5. 认证成功后可进行游戏操作

**重要说明**:
- 未认证的连接可以发送：`LOGIN`、`GET_GAME_CONFIG` 消息
- 游戏操作相关消息需要先完成认证
- 超过30秒未认证的连接将被自动断开
- 游戏配置不再自动推送，需要客户端主动请求

### 4. JWT Token刷新机制 ✅ *已实现*

系统提供两种Token刷新方式：

#### 4.1 HTTP API 刷新（推荐）
**端点**: `POST /v1/auth/refresh`

**请求头**:
```
Authorization: Bearer <current_token>
```

**响应示例**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 7200,
  "expires_at": "2024-01-20T19:30:00Z"
}
```

#### 4.2 WebSocket 自动刷新
- WebSocket 连接会在令牌过期前30分钟自动刷新
- 刷新消息类型：`TOKEN_REFRESH`
- 客户端需要保存新令牌用于后续重连

### 5. 消息格式

所有消息使用以下封装格式（已优化字段名以减少传输大小）：

```json
{
  "i": "msg_123456",
  "t": "PLACE_BET",
  "p": { ... }
}
```

**字段说明**：
- `i` (id): 消息ID
- `t` (type): 消息类型
- `p` (payload): 消息负载（用于请求/响应）
- `d` (data): 数据字段（用于推送消息）

**注意**：为了减少传输数据量，已移除timestamp字段

### 6. 事件类型枚举 ✅ *已实现*

WebSocket 系统使用 Protocol Buffers 枚举来定义所有事件类型，提供类型安全和更好的开发体验。

#### EventType 枚举定义

```protobuf
enum EventType {
  EVENT_TYPE_UNSPECIFIED = 0;           // 未指定类型
  EVENT_TYPE_BET_PLACED = 1;            // 下注事件
  EVENT_TYPE_GAME_RESULT = 2;           // 游戏结果
  EVENT_TYPE_PLAYER_JOINED = 3;         // 玩家加入
  EVENT_TYPE_PLAYER_LEFT = 4;           // 玩家离开
  EVENT_TYPE_BIG_WIN = 5;               // 大额获胜
  EVENT_TYPE_JACKPOT = 6;               // 累积奖池
  EVENT_TYPE_BET_ACTIVITY = 7;          // 单个投注活动
  EVENT_TYPE_BET_ACTIVITY_BATCH = 8;    // 批量投注活动
  EVENT_TYPE_LIVE_STATS = 9;            // 实时统计
  EVENT_TYPE_BALANCE_UPDATE = 10;       // 余额更新通知 ✅ *新增*
}
```

#### 事件类型说明

| 事件类型 | 值 | 描述 | 自动订阅 |
|---------|---|------|---------|
| EVENT_TYPE_BET_ACTIVITY_BATCH | 8 | 批量投注活动推送 | ✅ 是 |
| EVENT_TYPE_GAME_RESULT | 2 | 游戏结果事件 | ❌ 否 |
| EVENT_TYPE_BIG_WIN | 5 | 大额获胜通知 | ❌ 否 |
| EVENT_TYPE_JACKPOT | 6 | 累积奖池中奖 | ❌ 否 |
| EVENT_TYPE_LIVE_STATS | 9 | 实时统计更新 | ❌ 否 |
| EVENT_TYPE_BALANCE_UPDATE | 10 | 余额变化通知 | ✅ 是* |

**注意**：
- EVENT_TYPE_BET_ACTIVITY_BATCH：所有客户端连接后自动接收
- EVENT_TYPE_BALANCE_UPDATE*：认证后自动推送，无需手动订阅

### 7. 消息类型

#### 1. 登录认证（新增）
**类型**: `LOGIN`

**描述**: 在WebSocket连接建立后进行身份认证

**请求消息**:
```json
{
  "i": "msg_login_123",
  "t": "LOGIN",
  "p": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**响应类型**: `LOGIN_RESPONSE`

**成功响应**:
```json
{
  "i": "msg_login_resp_123",
  "t": "LOGIN_RESPONSE",
  "p": {
    "success": true,
    "userId": "12345",
    "gameId": "inhousegame:dice",
    "sessionId": "sess_abc123"
  }
}
```

**失败响应**:
```json
{
  "i": "msg_login_resp_123",
  "t": "LOGIN_RESPONSE",
  "p": {
    "success": false,
    "error": {
      "code": "INVALID_TOKEN",
      "message": "Token is invalid or expired"
    }
  }
}
```

**注意事项**:
- LOGIN消息必须在连接建立后30秒内发送
- 认证成功后才能发送游戏相关请求
- 游戏配置需要通过 GET_GAME_CONFIG 主动请求（不再自动推送）
- 认证成功后，服务器会发送 INITIALIZATION_COMPLETE 消息

#### 2. 下注请求
**类型**: `PLACE_BET_REQUEST`

**参数说明**:
- `client_seed`: 客户端种子
  - 来源：由前端客户端生成的随机字符串
  - 用途：用于可证明公平机制，确保游戏结果的公正性
  - 建议格式：包含时间戳的随机字符串，如 "my-seed-" + Date.now()
  - 长度：建议 10-50 个字符

**完整请求消息**:
```json
{
  "i": "msg_123456",
  "t": "PLACE_BET",
  "p": {
    "amount": "100.50000000",
    "gameParams": {
      "dice": {
        "target": 50,
        "isRollOver": true
      }
    },
    "clientSeed": "my-lucky-seed-123"
  }
}
```

> ⚠️ **更新说明**: `game_type` 字段已被移除，游戏类型现在通过 `gameParams` 的结构自动识别。

**完整响应消息**:
```json
{
  "i": "msg_123457",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "betId": "bet_789",
    "gameResult": {
      "gameId": "game_456",
      "betAmount": "100.50000000",
      "winAmount": "201.00000000",
      "isWin": true,
      "gameOutcome": {
        "diceOutcome": {
          "roll": "75.23",
          "target": "50.00",
          "isRollOver": true
        }
      },
      "multiplier": "2.00000000",
      "timestamp": 1640995200000
    },
    "balance": "900.00000000"
  }
}

> 💡 **技术说明**：所有布尔类型字段（如 `isWin`）即使值为 `false` 也会在响应中返回。这是通过设置 `protojson.MarshalOptions.EmitUnpopulated = true` 实现的，确保客户端能正确接收所有字段。
```

> ✅ **注意**：余额信息不再返回，由聚合器管理。

#### 2. 获取游戏状态
**类型**: `GET_GAME_STATE_REQUEST`

**完整请求消息**:
```json
{
  "i": "msg_123458",
  "t": "GET_GAME_STATE",
  "p": {}
}
```

**完整响应消息**:
```json
{
  "i": "msg_123459",
  "t": "GET_GAME_STATE_RESPONSE",
  "p": {
    "balance": "1000.00000000",
    "gameState": "{\"lastBet\": {...}, \"statistics\": {...}}",
    "serverSeedInfo": {
      "hashedServerSeed": "current-hash",
      "currentNonce": 43,
      "createdAt": 1640990000000
    }
  }
}
```

#### 3. 游戏配置

##### 主动请求（无需登录）
**请求类型**: `GET_GAME_CONFIG`
**响应类型**: `GET_GAME_CONFIG_RESPONSE`
**描述**: 客户端可以在连接建立后立即请求游戏配置，无需先登录。适用于前端需要提前渲染游戏界面的场景。

**请求消息（获取单个游戏）**:
```json
{
  "i": "msg_config_123",
  "t": "GET_GAME_CONFIG",
  "p": {
    "gameId": "inhousegame:dice"  // 指定游戏ID
  }
}
```

**请求消息（获取所有活跃游戏）**:
```json
{
  "i": "msg_config_123",
  "t": "GET_GAME_CONFIG",
  "p": {
    "allGames": true  // 获取所有活跃游戏
  }
}
```

**响应消息**:
```json
{
  "i": "msg_config_resp_123",
  "t": "GET_GAME_CONFIG_RESPONSE",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:dice",
        "config": {  // 使用 google.protobuf.Any 类型包装的配置
          "id": 2000003,
          "gameId": "inhousegame:dice",
          "gameName": "Dice",
          "category": "instant",
          "status": "active",
          "description": "Classic dice game with adjustable win probability",
          "thumbnail": "/games/dice/thumbnail.png",
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",  // 币种类型
              "defaultBet": 0.2,
              "minBet": 0.1,
              "maxBet": 400000
            }
          ],
          "minBet": 0.0000001,
          "maxBet": 20000000000,
          "rtp": 97.0,
          "features": ["provably_fair", "instant_play", "auto_play"],
          "defaultRtp": "97%",
          "betRange": "0.0000001,20000000000",
          "maxRewardMultiplier": 5000000000
          // ... 更多配置字段（rtpOptions, newUser, oldUser, totalControl）
        }
      }
    ]
  }
}
```

**使用场景**:
1. **预加载配置**: 连接建立后立即请求，用于提前渲染游戏UI
2. **配置刷新**: 游戏运行中动态刷新配置
3. **游戏切换**: 切换游戏时获取新游戏配置

**重要说明**:
- GET_GAME_CONFIG 无需登录即可使用
- 游戏配置不再自动推送，需要客户端主动请求
- 响应采用完全结构化的配置格式
- 不同游戏类型使用各自的结构化配置消息（dice_config、mines_config 等）
- 配置包含游戏的所有参数，包括基本信息、下注限制、RTP配置等
- 前端应该根据这些配置初始化游戏界面

**配置格式说明** ✅ *更新*: 
- 从 2025年1月起，GameConfig 使用 `google.protobuf.Any` 类型包装配置
- 这使得系统可以支持任意类型的游戏配置，无需预先定义所有游戏类型
- Dice 游戏：配置被包装为 `DiceGameConfig` 消息
- 其他游戏类型：可以动态添加新的配置类型，无需修改 proto 定义
- 客户端需要根据 `gameId` 解析对应的配置类型

**Dice 游戏配置结构（config 字段内容）**:
```typescript
interface DiceGameConfig {
  // 基本信息
  id: number;                    // 游戏ID
  gameId: string;               // 游戏标识，如 "inhousegame:dice"
  gameName: string;             // 游戏名称，如 "Dice Game"
  category: string;             // 游戏类别，如 "instant"
  status: string;               // 游戏状态，如 "active"
  description: string;          // 游戏描述
  thumbnail: string;            // 缩略图URL
  
  // 下注信息（支持多币种）
  betInfo: Array<{
    currency: string;           // 货币代码，如 "USD", "BTC"
    currencyType: string;       // 币种类型："fiat"（法币）或 "crypto"（加密货币）
    defaultBet: number;         // 默认下注金额
    minBet: number;            // 最小下注金额
    maxBet: number;            // 最大下注金额
    maxProfit?: number;        // 最大赢利限制
  }>;
  
  // 游戏特性
  features?: string[];          // 如 ["provably_fair", "instant_play", "auto_play"]
  
  // RTP 配置
  defaultRTP: string;          // 默认 RTP，如 "98"
  betRange: string;            // 下注范围，如 "0.1-1000"
  maxRewardMultiplier: number; // 最大奖励倍数，如 99
  
  // RTP 选项
  rtpOptions: {
    newOldUserDefault: string; // 新老用户默认 RTP
    selectable: string;        // 可选 RTP 值，逗号分隔，如 "96,97,98"
  };
  
  // 新用户配置
  newUser: {
    betTimesOptions: string;   // 下注次数选项，如 "10,20,30"
    defaultRTP: string;        // 新用户默认 RTP
    rtpRandomInterval: number; // RTP 随机区间
  };
  
  // 老用户配置
  oldUser: {
    rtpRandomInterval: number; // RTP 随机区间
  };
  
  // 总控制配置
  totalControl: {
    minBetUSD: number;         // 最小下注金额（美元）
    isStableRTP: number;       // 是否稳定 RTP (0/1)
    rtpRange: string;          // RTP 范围，如 "96-98"
  };
  
  // Dice 特有参数
  gameParameters?: {
    rollSliderRange: {         // 滚动滑块范围
      min: number;
      max: number;
    };
    rollNumberRange: {         // 滚动数字范围
      min: number;
      max: number;
    };
  };
}
```

**币种类型说明**：
- `currencyType`: 用于标识货币类型，便于前端进行分类展示和处理
  - `"fiat"`: 法定货币（如 USD、EUR、CNY 等）
  - `"crypto"`: 加密货币（如 BTC、ETH、USDT 等）
- 该字段现在是必填字段，所有货币都有明确的类型标识

**其他游戏配置结构**:
其他游戏（如 Mines、Blackjack）的配置结构类似，但会有各自特定的游戏参数。基本结构包含：
- 基本信息（id、gameId、gameName 等）
- 下注限制（betInfo 数组）
- RTP 配置
- 游戏特定参数




#### 4. 订阅事件
**类型**: `SUBSCRIBE_REQUEST`

**完整请求消息**:
```json
{
  "i": "msg_123460",
  "t": "SUBSCRIBE",
  "p": {
    "eventTypes": ["EVENT_TYPE_GAME_RESULT", "EVENT_TYPE_LIVE_STATS", "EVENT_TYPE_JACKPOT"],
    "filters": "{\"gameTypes\": [\"dice\", \"crash\"]}"
  }
}
```

**完整响应消息**:
```json
{
  "i": "msg_123461",
  "t": "SUBSCRIBE_RESPONSE",
  "p": {
    "subscriptionId": "sub_123",
    "eventTypes": ["EVENT_TYPE_GAME_RESULT", "EVENT_TYPE_LIVE_STATS", "EVENT_TYPE_JACKPOT"],
    "success": true,
    "message": "成功订阅事件"
  }
}
```

#### 5. 游戏事件（服务器推送）
**类型**: `GAME_EVENT`

**完整事件消息**:
```json
{
  "i": "msg_123462",
  "t": "GAME_EVENT",
  "p": {
    "eventId": "evt_999",
    "eventType": "EVENT_TYPE_GAME_RESULT",
    "timestamp": 1640995300000,
    "gameResult": {
      "gameId": "inhousegame:dice",
      "playerId": "player_123",
      "betAmount": "500.00000000",
      "winAmount": "1000.00000000",
      "isWin": true,
      "multiplier": "2.00000000",
      "gameOutcome": {
        "diceOutcome": {
          "roll": "25.45",
          "target": "50.00",
          "isRollOver": false
        }
      },
      "timestamp": 1640995300000
    }
  }
}
```

#### 6. 实时投注活动推送 ✅ *已实现*
**类型**: `GAME_EVENT` (event_type: `EVENT_TYPE_BET_ACTIVITY_BATCH`)

**描述**: 实时推送其他玩家的投注活动，营造热闹的游戏氛围。所有客户端连接后自动订阅此事件。

**BetActivityEvent 结构**:
```protobuf
message BetActivityEvent {
  string masked_player_id = 1;    // 脱敏的玩家ID (如 "pla***123")
  string bet_amount = 2;          // 下注金额 (字符串，保留8位小数)
  string currency = 3;            // 货币代码
  string potential_win = 4;       // 潜在赢利 (字符串，保留8位小数)
  string multiplier = 5;          // 倍数 (字符串，保留8位小数)
  oneof result {                  // 游戏结果 (可选)
    bool is_win = 6;              // 是否获胜
  }
  string win_amount = 7;          // 获胜金额 (仅在获胜时有值)
  int64 timestamp = 8;            // Unix时间戳
  string country_code = 9;        // 国家代码 (可选，如 "US")
}
```

**BetActivityBatch 结构**:
```protobuf
message BetActivityBatch {
  repeated BetActivityEvent activities = 1;  // 投注活动列表
  int32 total_count = 2;                    // 时间段内总投注数
  int32 sampled_count = 3;                  // 采样后包含的投注数
  int64 period_start = 4;                   // 时间段开始 (Unix时间戳)
  int64 period_end = 5;                     // 时间段结束 (Unix时间戳)
  int64 sequence = 6;                       // 批次序列号
}
```

**推送消息示例**:
```json
{
  "i": "msg_event_123",
  "t": "GAME_EVENT",
  "p": {
    "eventId": "evt_789",
    "eventType": "EVENT_TYPE_BET_ACTIVITY_BATCH",
    "timestamp": 1640995300000,
    "betActivityBatch": {
      "activities": [
        {
          "maskedPlayerId": "pla***567",
          "betAmount": "100.00000000",
          "currency": "USD",
          "potentialWin": "200.00000000",
          "multiplier": "2.00000000",
          "timestamp": 1640995295000,
          "countryCode": "US",
          "gameId": "inhousegame:dice"
        },
        {
          "maskedPlayerId": "abc***123",
          "betAmount": "50.00000000",
          "currency": "USD",
          "potentialWin": "500.00000000",
          "multiplier": "10.00000000",
          "isWin": true,
          "winAmount": "500.00000000",
          "timestamp": 1640995298000,
          "countryCode": "CN",
          "gameId": "inhousegame:dice"
        }
      ],
      "totalCount": 15,
      "sampledCount": 2,
      "periodStart": 1640995290000,
      "periodEnd": 1640995300000,
      "sequence": 42
    }
  }
}
```

**实现特性**:
1. **自动订阅**: 客户端连接后自动订阅，无需手动订阅
2. **批量发送**: 每 500ms 批量发送一次，减少网络开销
3. **智能采样**: 
   - 小额投注（< $10）：10% 显示概率
   - 中额投注（$10-$100）：50% 显示概率
   - 大额投注（> $100）：100% 显示
   - 10倍以上大赢：始终显示
4. **隐私保护**: 玩家ID脱敏显示，如 "pla***123"
5. **队列缓冲**: 5000条消息缓冲，防止突发流量

**客户端处理示例**:
```javascript
// 监听投注活动事件
socket.on('message', (data) => {
  if (data.t === 'GAME_EVENT' && 
      data.p.eventType === 'EVENT_TYPE_BET_ACTIVITY_BATCH') {
    const batch = data.p.betActivityBatch;
    
    // 处理每个投注活动
    batch.activities.forEach(activity => {
      if (activity.isWin) {
        // 显示获胜动画
        showWinAnimation(activity.maskedPlayerId, activity.winAmount);
      } else {
        // 显示投注信息
        showBetActivity(activity.maskedPlayerId, activity.betAmount);
      }
    });
  }
});
```

#### 7. 错误消息
**类型**: `ERROR`

**完整错误消息**:
```json
{
  "i": "msg_123463",
  "t": "ERROR",
  "p": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "您的余额不足以进行此下注",
    "details": "{\"required\": \"100.00\", \"available\": \"50.00\"}",
    "requestId": "msg_123456"
  }
}
```

#### 8. 获取余额 ✨
**类型**: `GET_BALANCE`

**描述**: 查询玩家余额，支持多币种查询

**请求消息（使用默认币种）**:
```json
{
  "i": "msg_balance_123",
  "t": "GET_BALANCE",
  "p": {}
}
```

**请求消息（指定币种）**:
```json
{
  "i": "msg_balance_123",
  "t": "GET_BALANCE",
  "p": {
    "currency": "EUR"  // 可选，不提供则使用会话默认币种
  }
}
```

**响应类型**: `GET_BALANCE_RESPONSE`

**响应消息**:
```json
{
  "i": "msg_balance_resp_123",
  "t": "GET_BALANCE_RESPONSE",
  "p": {
    "balance": "1000.00000000",
    "currency": "EUR"  // 返回查询的币种
  }
}
```

**参数说明**:
- `currency` (可选): 要查询的币种代码（如 "USD", "EUR", "BTC"）
  - 如果不提供，使用 JWT token 中的默认币种
  - 如果提供，查询指定币种的余额

**使用场景**:
1. **默认查询**: 不传币种参数，查询会话默认币种余额（最常见）
2. **多币种查询**: 传入特定币种，查询该币种余额（如玩家有多个钱包）

**注意事项**:
- 必须先通过 LOGIN 认证才能查询余额
- 余额从聚合器（io）实时获取
- balance 字段为字符串格式，保留 8 位小数
- 支持查询与会话币种不同的其他币种余额

#### 9. 余额更新通知 ✅ *新增*
**类型**: `BALANCE_UPDATE`

**描述**: 服务器主动推送余额变化通知，当系统检测到玩家余额发生变化时自动推送

**推送消息示例**:
```json
{
  "i": "msg_balance_update_123",
  "t": "BALANCE_UPDATE",
  "p": {
    "balance": "950.00000000",
    "currency": "USD",
    "timestamp": 1640995300000
  }
}
```

**BalanceUpdateEvent 结构**:
```protobuf
message BalanceUpdateEvent {
  string balance = 1;    // 更新后的余额（字符串格式，保留8位小数）
  string currency = 2;   // 货币代码
  int64 timestamp = 3;   // 更新时间戳（Unix时间戳）
}
```

**触发条件**:
1. **定时同步**: 每30秒自动同步一次余额
2. **下注前强制同步**: 在玩家下注前触发
3. **结算后同步**: 游戏结算完成后立即同步
4. **余额变化检测**: 当检测到余额与缓存不一致时推送

**实现特性**:
- **智能定时器**: 强制同步会重置定时器，避免重复同步
- **变化检测**: 只有余额真正发生变化时才推送通知
- **最小间隔保护**: 两次同步之间至少间隔2秒，防止API过载
- **自动启动**: 玩家登录成功后自动启动余额同步器

**客户端处理示例**:
```javascript
// 监听余额更新事件
socket.on('message', (data) => {
  if (data.t === 'BALANCE_UPDATE') {
    const { balance, currency, timestamp } = data.p;
    
    // 更新UI显示的余额
    updateBalanceDisplay(balance, currency);
    
    // 记录更新时间
    console.log(`Balance updated at ${new Date(timestamp)}: ${balance} ${currency}`);
  }
});
```

**注意事项**:
- 此事件为服务器主动推送，客户端无需订阅
- 余额同步依赖聚合器（io）的实时数据
- 在 v1.0 架构下，余额由聚合器管理，游戏服务器只负责同步和推送
- 首次同步（如登录后第一次）不会触发推送，避免重复通知

#### 10. 心跳保活
**机制**: 使用数字 `0` 和 `1` 作为心跳消息

**说明**:
- 客户端应定期（建议每30-45秒）发送数字 `0` 作为心跳ping
- 服务端收到 `0` 后会立即回复数字 `1`
- 如果服务端60秒内没有收到任何消息（包括心跳），连接将被关闭

**示例**:
- 客户端发送: `0`
- 服务端回复: `1`

**注意**:
- 心跳消息是纯文本格式，不是JSON
- 心跳由客户端主动发起，服务端被动响应
- 建议客户端在网络空闲时定期发送心跳

## 游戏前端 API

### 基础 URL
`https://dev.hicasino.xyz/v1/frontend`

### 认证
所有请求需要 `Authorization` 头：
```
Authorization: Bearer <token>
```

**JWT 认证说明** ✅ *已实现*:
- Game API 端点（`/api/game/v1/*`）使用 JWT Bearer token 认证
- Token 从 Authorization header 中提取：`Authorization: Bearer <token>`
- JWT 中包含用户信息（player_id、user_id、aggregator_id 等）
- 用户信息自动从 token 中提取，无需在请求参数中传递敏感信息

### 认证端点

#### 刷新Token
**POST** `/v1/auth/refresh`

刷新即将过期的JWT令牌。建议在令牌过期前30分钟调用此接口。

**请求头**:
```
Authorization: Bearer <current_token>
```

**响应**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 7200,
  "expires_at": "2025-07-12T10:00:00Z"
}
```

**错误响应**:
```json
{
  "code": "TOKEN_EXPIRED",
  "message": "Token has expired"
}
```


### 骰子游戏端点

#### 获取骰子游戏配置
**GET** `/games/dice/config`

**响应**:
```json
{
  "success": true,
  "data": {
    "min_bet": 1.0,
    "max_bet": 10000.0,
    "house_edge": 1.0,
    "min_target": 1.0,
    "max_target": 99.0
  }
}
```

#### 创建服务器种子
**POST** `/v1/games/dice/server-seed`

**请求体**:
```json
{}
```

> 注意：用户信息从 JWT Token 中自动获取，无需在请求体中传递。

**响应**:
```json
{
  "success": true,
  "data": {
    "hashed_server_seed": "sha256-hash-of-server-seed",
    "current_nonce": 0
  }
}
```

#### 下注骰子游戏
**POST** `/games/dice/bet`

**请求体**:
```json
{
  "amount": 100.50,
  "currency": "USD",
  "target": 50.0,
  "is_roll_over": true,
  "client_seed": "my-lucky-seed"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "bet_id": "bet_123",
    "state": "completed",
    "roll": 75.23,
    "target": 50.0,
    "is_roll_over": true,
    "win_amount": 201.00,
    "is_win": true,
    "multiplier": 2.0,
    "provably_fair": {
      "client_seed": "my-lucky-seed",
      "server_seed": "",  // 活跃种子不会立即返回原始值
      "hashed_server_seed": "sha256-hash",
      "nonce": 42,
      "seed_revealed": false  // 种子未揭示
    }
  }
}
```

> ⚠️ **安全更新**：为了防止种子预测攻击，活跃的服务端种子不会在投注后立即返回。种子只有在轮换后才会揭示原始值。

#### 轮换服务器种子 ✅ *新增*
**POST** `/v1/games/dice/rotate-seed`

轮换当前的服务器种子，创建新种子并揭示旧种子的原始值。

**请求体**:
```json
{}
```

> 注意：用户信息从 JWT Token 中自动获取，无需在请求体中传递。

**响应**:
```json
{
  "success": true,
  "data": {
    "old_seed": {
      "seed_id": 12345,
      "seed_value": "revealed-old-seed-value",  // 旧种子的原始值
      "seed_hash": "sha256-hash-of-old-seed",
      "total_bets": 42,
      "last_nonce": 41,
      "is_active": false,
      "created_at": 1640990000000,
      "revealed_at": 1640995200000
    },
    "new_seed": {
      "seed_id": 12346,
      "seed_value": "",  // 新种子不返回原始值
      "seed_hash": "sha256-hash-of-new-seed",
      "total_bets": 0,
      "last_nonce": 0,
      "is_active": true,
      "created_at": 1640995200000,
      "revealed_at": 0
    }
  }
}
```

**使用场景**:
- 玩家希望验证之前的游戏结果
- 定期轮换种子以增强安全性
- 在提现前轮换种子

#### 获取下注历史
**GET** `/games/dice/history/{player_id}?limit=20&offset=0`

> ⚠️ **已废弃**：此接口使用 player_id 查询，建议使用 `/api/game/v1/history` 新接口，通过 JWT 中的 UserID 查询。

**响应**:
```json
{
  "success": true,
  "data": {
    "total": 150,
    "offset": 0,
    "limit": 20,
    "bets": [
      {
        "bet_id": "bet_123",
        "bet_amount": 100.50,
        "win_amount": 201.00,
        "is_win": true,
        "created_at": 1640995200,
        "game_outcome": {
          "roll": 75.23,
          "target": 50.0,
          "is_roll_over": true,
          "multiplier": 2.0
        },
        "provably_fair": {
          "client_seed": "my-lucky-seed",
          "server_seed": "revealed-server-seed",  // 只有非活跃种子才返回原始值
          "hashed_server_seed": "sha256-hash",
          "nonce": 42,
          "seed_revealed": true  // 种子已揭示
        }
      }
    ]
  }
}
```

### 地雷(Mines)游戏端点

#### 获取地雷游戏配置
**GET** `/api/v1/mines/config`

**响应**:
```json
{
  "success": true,
  "data": {
    "min_bet_amount": 1.0,
    "max_bet_amount": 10000.0,
    "available_mines": [1, 3, 5, 10, 15, 20, 24],
    "house_edge": 1.0
  }
}
```

#### 开始新游戏
**POST** `/api/v1/mines/bet`

**请求体**:
```json
{
  "amount": 100.0,
  "currency": "USD",
  "mines_count": 5,
  "client_seed": "player-chosen-seed"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "game_id": "mines_abc123",
    "request_id": "req_456",
    "game_state": {
      "game_id": "mines_abc123",
      "status": "STATUS_IN_PROGRESS",
      "bet_amount": 100.0,
      "mines_count": 5,
      "revealed_tiles": [],
      "safe_tiles_revealed": 0,
      "created_at": 1640995200000,
      "updated_at": 1640995200000
    },
    "provably_fair": {
      "client_seed": "player-chosen-seed",
      "hashed_server_seed": "hash_of_server_seed",
      "nonce": 1
    }
  }
}
```

#### 揭示瓦片
**POST** `/api/v1/mines/reveal`

**请求体**:
```json
{
  "game_id": "mines_abc123",
  "tile_index": 12
}
```

**响应（安全瓦片）**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_789",
    "is_mine": false,
    "game_state": {
      "game_id": "mines_abc123",
      "status": "STATUS_IN_PROGRESS",
      "bet_amount": 100.0,
      "mines_count": 5,
      "revealed_tiles": [12],
      "safe_tiles_revealed": 1
    },
    "current_multiplier": 1.04,
    "next_multiplier": 1.09
  }
}
```

**响应（触雷）**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_790",
    "is_mine": true,
    "game_state": {
      "game_id": "mines_abc123",
      "status": "STATUS_LOST",
      "bet_amount": 100.0,
      "mines_count": 5,
      "revealed_tiles": [12, 15],
      "safe_tiles_revealed": 1
    },
    "result": {
      "mine_positions": [2, 5, 8, 15, 20],
      "safe_tiles_revealed": 1,
      "final_multiplier": 0,
      "payout": 0,
      "provably_fair": {
        "client_seed": "player-chosen-seed",
        "server_seed": "revealed-server-seed",
        "hashed_server_seed": "hash_of_server_seed",
        "nonce": 1
      }
    }
  }
}
```

#### 现金提取
**POST** `/api/v1/mines/cashout`

**请求体**:
```json
{
  "game_id": "mines_abc123",
  "player_id": "player123"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_791",
    "payout": 208.0,
    "game_state": {
      "game_id": "mines_abc123",
      "status": "STATUS_CASHED_OUT",
      "bet_amount": 100.0,
      "mines_count": 5,
      "revealed_tiles": [3, 7, 12, 18, 22],
      "safe_tiles_revealed": 5
    },
    "result": {
      "mine_positions": [2, 5, 8, 15, 20],
      "safe_tiles_revealed": 5,
      "final_multiplier": 2.08,
      "payout": 208.0,
      "provably_fair": {
        "client_seed": "player-chosen-seed",
        "server_seed": "revealed-server-seed",
        "hashed_server_seed": "hash_of_server_seed",
        "nonce": 1
      }
    }
  }
}
```

#### 获取游戏状态
**GET** `/api/v1/mines/state/{game_id}`

**响应**:
```json
{
  "success": true,
  "data": {
    "game_state": {
      "game_id": "mines_abc123",
      "status": "STATUS_IN_PROGRESS",
      "bet_amount": 100.0,
      "mines_count": 5,
      "revealed_tiles": [3, 7, 12],
      "safe_tiles_revealed": 3
    },
    "current_multiplier": 1.13,
    "next_multiplier": 1.19
  }
}
```

#### WebSocket 消息格式（Mines）

**开始游戏（使用通用 PlaceBetRequest）**:
```json
{
  "id": "msg_123",
  "type": "PLACE_BET_REQUEST",
  "timestamp": 1640995200000,
  "place_bet_request": {
    "game_type": "mines",
    "amount": 100.0,
    "currency": "USD",
    "game_params": {
      "mines": {
        "mines_count": 5
      }
    },
    "client_seed": "player-chosen-seed"
  }
}
```

**注意**: player_id 无需在消息中传递，由 WebSocket 认证自动关联

**揭示瓦片请求（通过 data 字段）**:
```json
{
  "id": "msg_124",
  "type": "mines_reveal_tile",
  "timestamp": 1640995200000,
  "data": "{\"game_id\": \"mines_abc123\", \"tile_index\": 12}"
}
```

**现金提取请求（通过 data 字段）**:
```json
{
  "id": "msg_125",
  "type": "mines_cash_out",
  "timestamp": 1640995200000,
  "data": "{\"game_id\": \"mines_abc123\"}"
}
```

### 21点(Blackjack)游戏端点

#### 获取21点游戏配置
**GET** `/v1/blackjack/config`

**响应**:
```json
{
  "success": true,
  "data": {
    "min_bet_amount": 10.0,
    "max_bet_amount": 5000.0,
    "blackjack_payout_ratio": 1.5,
    "insurance_payout_ratio": 2.0,
    "split_allowed": true,
    "double_after_split_allowed": true,
    "max_splits": 3,
    "surrender_allowed": false,
    "house_edge": 0.5
  }
}
```

#### 下注并开始游戏
**POST** `/v1/blackjack/bet`

**请求体**:
```json
{
  "amount": 100.0,
  "currency": "USD",
  "client_seed": "player-chosen-seed"
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_456",
    "game_id": "bj_abc123",
    "game_state": {
      "game_id": "bj_abc123",
      "status": "STATUS_PLAYER_TURN",
      "bet_amount": 100.0,
      "dealer_hand": {
        "cards": [
          {"suit": "SUIT_HEARTS", "rank": "RANK_ACE"},
          {"suit": "SUIT_CLUBS", "rank": "RANK_TEN", "is_face_down": true}
        ],
        "soft_total": 11,
        "hard_total": 11,
        "is_soft": true
      },
      "player_hands": [{
        "cards": [
          {"suit": "SUIT_DIAMONDS", "rank": "RANK_KING"},
          {"suit": "SUIT_SPADES", "rank": "RANK_SEVEN"}
        ],
        "soft_total": 17,
        "hard_total": 17,
        "is_soft": false,
        "hand_id": "hand_main"
      }],
      "current_hand_index": 0,
      "insurance_offered": true
    },
    "provably_fair": {
      "client_seed": "player-chosen-seed",
      "hashed_server_seed": "hash_of_server_seed",
      "nonce": 1
    },
    "insurance_available": true
  }
}
```

#### 玩家动作
**POST** `/v1/blackjack/action`

**请求体**:
```json
{
  "game_id": "bj_abc123",
  "action": "ACTION_HIT",
  "hand_id": "hand_main"
}
```

**动作类型**:
- `ACTION_HIT`: 要牌
- `ACTION_STAND`: 停牌
- `ACTION_DOUBLE`: 加倍
- `ACTION_SPLIT`: 分牌

**响应（继续游戏）**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_457",
    "game_state": {
      "game_id": "bj_abc123",
      "status": "STATUS_PLAYER_TURN",
      "player_hands": [{
        "cards": [
          {"suit": "SUIT_DIAMONDS", "rank": "RANK_KING"},
          {"suit": "SUIT_SPADES", "rank": "RANK_SEVEN"},
          {"suit": "SUIT_HEARTS", "rank": "RANK_THREE"}
        ],
        "soft_total": 20,
        "hard_total": 20,
        "is_soft": false,
        "hand_id": "hand_main"
      }]
    },
    "action_completed": true
  }
}
```

**响应（游戏结束）**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_458",
    "game_state": {
      "game_id": "bj_abc123",
      "status": "STATUS_FINISHED",
      "dealer_hand": {
        "cards": [
          {"suit": "SUIT_HEARTS", "rank": "RANK_ACE"},
          {"suit": "SUIT_CLUBS", "rank": "RANK_TEN"}
        ],
        "soft_total": 21,
        "hard_total": 21,
        "is_blackjack": true
      }
    },
    "action_completed": true,
    "result": {
      "hand_results": [{
        "hand_id": "hand_main",
        "final_hand": {
          "cards": [...],
          "soft_total": 20,
          "hard_total": 20
        },
        "bet_amount": 100.0,
        "payout": 0,
        "outcome": "lose"
      }],
      "total_payout": 0,
      "dealer_blackjack": true,
      "dealer_final_hand": {...},
      "provably_fair": {
        "client_seed": "player-chosen-seed",
        "server_seed": "revealed-server-seed",
        "hashed_server_seed": "hash_of_server_seed",
        "nonce": 1
      }
    }
}
```

#### 保险决定
**POST** `/v1/blackjack/insurance`

**请求体**:
```json
{
  "game_id": "bj_abc123",
  "take_insurance": true
}
```

**响应**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_459",
    "game_state": {...},
    "insurance_accepted": true,
    "insurance_amount": 50.0
  }
}
```

#### 获取游戏状态
**GET** `/v1/blackjack/state/{game_id}`

**响应**:
```json
{
  "success": true,
  "data": {
    "game_state": {
      "game_id": "bj_abc123",
      "status": "STATUS_PLAYER_TURN",
      "bet_amount": 100.0,
      "dealer_hand": {...},
      "player_hands": [...],
      "current_hand_index": 0
    }
  }
}
```

#### WebSocket 消息格式（Blackjack）

**开始游戏（使用通用 PlaceBetRequest）**:
```json
{
  "id": "msg_200",
  "type": "PLACE_BET_REQUEST",
  "timestamp": 1640995200000,
  "place_bet_request": {
    "game_type": "blackjack",
    "amount": 100.0,
    "currency": "USD",
    "game_params": "{}",
    "client_seed": "player-chosen-seed"
  }
}
```

**玩家动作请求（通过 data 字段）**:
```json
{
  "id": "msg_201",
  "type": "BLACKJACK_PLAYER_ACTION",
  "timestamp": 1640995200000,
  "data": "{\"game_id\": \"bj_abc123\", \"action\": \"ACTION_HIT\", \"hand_id\": \"hand_main\"}"
}
```

**保险决定请求（通过 data 字段）**:
```json
{
  "id": "msg_202",
  "type": "BLACKJACK_INSURANCE",
  "timestamp": 1640995200000,
  "data": "{\"game_id\": \"bj_abc123\", \"take_insurance\": true}"
}
```

## 集成 API

> ⚠️ **实现状态**：
> - GameIntegrationService 的 proto 定义存在，但**无任何实现**
> - 无 API Key 认证机制
> - 无服务端处理逻辑
> - 未在 HTTP 服务器中注册
> - 以下为 API 设计规范，非实际可用接口
> 
> **注意**：如需集成游戏，请使用已实现的 Provider API 或 WebSocket API

### 基础 URL
`https://dev.hicasino.xyz/v1`

### 认证（未实现）
集成 API 使用 API 密钥认证：
```
X-API-Key: <integration-api-key>
```

### 端点

#### 列出可用游戏
**GET** `/games?type=dice`

**响应**:
```json
{
  "success": true,
  "data": {
    "games": [
      {
        "id": "inhousegame:dice",
        "name": "经典骰子",
        "description": "掷出大于或小于目标值来获胜",
        "type": "dice",
        "min_bet": 1.0,
        "max_bet": 10000.0,
        "thumbnail_url": "https://cdn.invoker.com/games/dice.png",
        "is_available": true
      }
    ]
  }
}
```

#### 获取游戏详情
**GET** `/games/{game_id}`

**响应**:
```json
{
  "success": true,
  "data": {
    "game": {
      "id": "inhousegame:dice",
      "name": "经典骰子",
      "description": "掷出大于或小于目标值来获胜",
      "type": "dice",
      "min_bet": 1.0,
      "max_bet": 10000.0,
      "thumbnail_url": "https://cdn.invoker.com/games/dice.png",
      "is_available": true
    },
    "options": {
      "house_edge": 1.0,
      "min_target": 1.0,
      "max_target": 99.0,
      "decimal_places": 2
    }
  }
}
```

#### 下注（集成）
**POST** `/bets`

**请求体**:
```json
{
  "game_id": "inhousegame:dice",
  "amount": 100.50,
  "options": {
    "target": 50.0,
    "is_roll_over": true
  },
  "client_seed": "player-chosen-seed"
}
```

**响应**:
```json
{
  "data": {
    "bet_id": "bet_789",
    "game_id": "inhousegame:dice",
    "amount": 100.50,
    "win_amount": 201.00,
    "is_win": true,
    "outcome": {
      "roll": 75.23,
      "target": 50.0,
      "is_roll_over": true,
      "multiplier": 2.0
    },
    "created_at": "2024-01-01T12:00:00Z",
    "provably_fair": {
      "client_seed": "player-chosen-seed",
      "server_seed": "revealed-server-seed",
      "nonce": 42
    }
  }
}
```

#### 获取下注历史（集成）
**GET** `/players/{player_id}/bets?game_id=dice_v1&page_size=20&page_token=`

**响应**:
```json
{
  "data": {
    "bets": [
      {
        "id": "bet_789",
            "game_id": "inhousegame:dice",
        "amount": 100.50,
        "win_amount": 201.00,
        "is_win": true,
        "outcome": { ... },
        "created_at": "2024-01-01T12:00:00Z",
        "provably_fair": { ... }
      }
    ],
    "next_page_token": "eyJvZmZzZXQiOjIwfQ=="
  }
}
```

#### 获取下注详情
**GET** `/bets/{bet_id}`

**响应**: 与历史记录中的单个下注相同

### 历史记录 API ✅ *已实现*

#### 查询玩家投注历史
**POST** `/v1/bets/page`

查询当前玩家的投注历史记录。支持分页、时间范围筛选和游戏筛选。

**认证要求**:
- 必须提供有效的 JWT token
- 使用 JWT token 中的 user_id（内部用户ID）进行查询
- **完全移除** player_id 支持（2025年1月更新）

> ✅ **安全升级**：从 2025年1月起，历史查询 API 已完全迁移到使用内部 UserID，不再接受外部 player_id。这提升了数据隔离性和安全性。

> 🔐 **公平性验证说明**：历史记录接口返回完整的公平性验证信息（`provablyFair` 字段），包括服务器种子、客户端种子、哈希值等。玩家可以使用这些信息验证游戏结果的公平性。实时投注响应中不再包含这些信息以优化性能。

**请求体**:
```json
{
  "gameId": "inhousegame:dice",  // 可选，筛选特定游戏
  "currency": "USD",             // 可选，筛选特定币种
  "startTime": "2025-01-01T00:00:00Z",  // 可选，开始时间
  "endTime": "2025-01-31T23:59:59Z",    // 可选，结束时间
  "page": {
    "page": 1,       // 页码，默认1
    "pageSize": 20   // 每页数量，默认20，最大100
  }
}
```

**响应**:
```json
{
  "bets": [
    {
      "bet": {
        "betId": "dice_20250101_001",
        "sessionId": "",
        "betAmount": {
          "amount": "10.00",
          "currency": "USD"
        },
        "winAmount": {
          "amount": "19.80",
          "currency": "USD"
        },
        "isWin": true,
        "multiplier": "1.98x",
        "createdAt": "2025-01-01T10:30:00Z"
      },
      "gameId": "inhousegame:dice",
      "gameName": "Dice",
      "gameData": {
        "target": 50.5,
        "rollResult": 45.23,
        "isRollOver": true
      },
      "provablyFair": {
        "serverSeed": "revealed_server_seed",
        "clientSeed": "player_chosen_seed",
        "nonce": 123
      }
    }
  ],
  "page": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 156,
    "totalPages": 8
  },
  "summary": {
    "totalBets": 156,
    "totalWagered": {
      "amount": "1560.00",
      "currency": "USD"
    },
    "totalWon": {
      "amount": "1432.50",
      "currency": "USD"
    },
    "netProfit": {
      "amount": "-127.50",
      "currency": "USD"
    },
    "winRate": 45.5
  }
}
```

**注意事项**:
- 系统使用 JWT token 中的 user_id（内部用户ID）进行查询
- 使用内部 user_id 确保完全的数据隔离和安全性
- 时间筛选使用 ISO 8601 格式
- 汇总统计（summary）基于当前筛选条件计算，不是全部历史

## 错误代码

### 通用错误代码

| 代码 | HTTP 状态 | 描述 |
|------|-------------|-------------|
| `INVALID_REQUEST` | 400 | 请求验证失败 |
| `UNAUTHORIZED` | 401 | 需要认证 |
| `FORBIDDEN` | 403 | 权限不足 |
| `NOT_FOUND` | 404 | 资源未找到 |
| `RATE_LIMITED` | 429 | 请求过多 |
| `INTERNAL_ERROR` | 500 | 服务器错误 |

### 游戏特定错误代码

| 代码 | 描述 |
|------|-------------|
| `INSUFFICIENT_BALANCE` | 玩家余额不足 |
| `BET_AMOUNT_TOO_LOW` | 下注金额低于最小值 |
| `BET_AMOUNT_TOO_HIGH` | 下注金额高于最大值 |
| `INVALID_GAME_PARAMS` | 游戏参数无效 |
| `GAME_NOT_AVAILABLE` | 游戏当前不可用 |
| `DUPLICATE_BET` | 检测到重复下注（幂等性） |

## 金额格式说明

### 重要更新 🔄
为了避免浮点数精度损失问题，所有涉及金额的字段已从 `double/float` 类型改为 `string` 类型。

### 金额字段规则

#### 1. 格式要求
- 所有金额必须以字符串形式传输
- 保留 8 位小数，如：`"123.45678901"`
- 不使用科学计数法
- 最小值：`"0.00000001"`
- 最大值：取决于具体业务限制

#### 2. 受影响的字段

**WebSocket API**:
- `amount`: 下注金额
- `balance`: 余额
- `bet_amount`: 下注金额
- `win_amount`: 获胜金额
- `multiplier`: 倍数
- `payout`: 支付金额
- `insurance_amount`: 保险金额
- `total_payout`: 总支付金额

**统计数据字段**:
- `total_volume`: 总交易量
- `total_winnings`: 总获胜金额
- `biggest_win`: 最大获胜金额
- `avg_bet_size`: 平均下注金额
- `jackpot_amount`: 奖池金额

#### 3. 示例

**请求示例**:
```json
{
  "type": "PLACE_BET",
  "payload": {
    "game_type": "dice",
    "amount": "10.12345678",  // 正确：字符串格式
    "game_params": {
      "target": 50.5
    }
  }
}
```

**响应示例**:
```json
{
  "type": "PLACE_BET_RESPONSE",
  "payload": {
    "bet_id": "bet_123",
    "balance": "990.00000000",      // 正确：保留 8 位小数
    "game_result": {
      "bet_amount": "10.12345678",  // 正确：与请求金额一致
      "win_amount": "20.24691356",  // 正确：精确计算
      "multiplier": "2.00000000",   // 正确：倍数也使用字符串
      "is_win": true
    }
  }
}
```

#### 4. 前端处理建议

**JavaScript 使用 decimal.js**:
```javascript
import Decimal from 'decimal.js';

// 解析金额
const amount = new Decimal(response.balance);

// 计算
const profit = amount.mul(multiplier.minus(1));

// 显示（保留 8 位小数）
document.getElementById('amount').textContent = amount.toFixed(8);

// 发送请求时
const betData = {
  amount: betAmount.toFixed(8)  // 转换为字符串
};
```

#### 5. 后端处理建议

**Go 使用 shopspring/decimal**:
```go
import "github.com/shopspring/decimal"

// 解析金额
amount, err := decimal.NewFromString(req.Amount)

// 计算
winAmount := betAmount.Mul(multiplier)

// 转回字符串（保留 8 位小数）
amountStr := amount.StringFixed(8)
```

### 迁移指南

1. **API 调用方**:
   - 更新请求参数，将数字类型金额转为字符串
   - 更新响应解析，处理字符串类型金额

2. **数据库存储**:
   - 继续使用 `DECIMAL(18,8)` 类型
   - 不需要修改数据库结构

3. **兼容性**:
   - 后端仍然兼容数字类型输入（会自动转换）
   - 建议尽快迁移到字符串格式

---

## Aggregator API

### 概述
Aggregator API 用于管理接入的游戏聚合器，包括创建聚合器、管理 API 密钥、配置 Webhook 等功能。

**基础信息**:
- **端点**: `https://dev.hicasino.xyz/api/aggregator/v1`
- **端口**: 8000（与主 HTTP 服务共用）
- **协议**: HTTP/HTTPS
- **认证**: 主密钥认证（用于管理操作）

**实现状态**:
- ✅ 所有 CRUD 操作已实现
- ✅ API 密钥加密存储
- ✅ Webhook 配置管理
- ✅ IP 白名单验证

### 主要功能

1. **创建聚合器**
2. **更新聚合器配置**
3. **管理 API 密钥**
4. **配置 Webhook**
5. **设置 IP 白名单**

详细的 Aggregator API 文档请参考专门的聚合器集成指南。

## 统一 Game API（设计阶段）

> ⚠️ **注意**：以下为统一 Game API 的设计方案，尚未实现。当前请继续使用 Provider API 和 WebSocket API。

### 设计理念

统一 Game API 旨在将现有的分散接口整合为一个清晰、一致的 API 结构：

- **统一的游戏接口**：所有游戏操作通过统一的端点访问
- **RESTful 设计**：遵循标准的 REST 原则
- **清晰的资源模型**：游戏、会话、投注等资源有明确的层次关系

### 计划的 API 结构

```
/v1/
├── game/          # 游戏相关接口
│   ├── games      # 游戏列表和配置
│   ├── sessions   # 会话管理
│   ├── bets       # 投注操作
│   └── history    # 历史记录
├── aggregator/    # 聚合器管理（已实现）
└── ws            # WebSocket 连接（已实现）
```

### 主要接口设计

#### 1. 游戏信息

##### 获取游戏列表
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

##### 获取游戏配置
**GET** `/v1/games/{game_id}/config`

获取特定游戏的详细配置。

#### 2. 会话管理

##### 创建游戏会话
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

#### 3. 投注操作

##### 下注
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

### 迁移计划

1. **第一阶段**：完成 API 设计和原型
2. **第二阶段**：实现核心功能，与现有 API 并行运行
3. **第三阶段**：迁移现有客户端到新 API
4. **第四阶段**：废弃旧 API 接口

### 优势

- **简化集成**：统一的接口设计降低学习成本
- **更好的扩展性**：便于添加新游戏和功能
- **标准化**：遵循 RESTful 最佳实践
- **类型安全**：使用 Protocol Buffers 提供强类型支持

## 相关文档
- [详细设计](./detailed-design-zh.md) - 架构和设计原则
- [序列图](./sequence-diagrams-zh.md) - 可视化流程展示
- [集成指南](others/integration-guide-zh.md) - 分步集成说明
- [聚合器集成指南](others/aggregator-integration-guide-zh.md) - 聚合器详细接入说明