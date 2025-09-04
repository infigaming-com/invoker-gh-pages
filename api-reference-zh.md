# API 参考 - Invoker Server

## 目录

1. [WebSocket API](#websocket-api) - 实时游戏通信
2. [错误代码](#错误代码)
3. [金额格式说明](#金额格式说明)

> **说明**：所有 RESTful API 接口文档已通过 Redoc 自动生成。本文档仅包含 WebSocket API 和相关格式说明。
> restful短连接接口文档地址：https://storage.googleapis.com/speedix-invoker-api-docs/index.html


## WebSocket API

### 消息格式说明

消息类型使用 `p` 字段（结构化对象）传递数据：

| 消息类型 | 格式 | 说明 |
|----------|------|------|
| INITIALIZATION_COMPLETE | `p` 字段（结构化对象） | 初始化完成通知 |
| GAME_CONFIG | `d` 字段（兼容） | 游戏配置 |
| gameOutcome | 结构化对象（oneof） | 游戏结果数据 |
| BALANCE_UPDATE | `p` 字段（结构化对象） | 余额更新通知 |

**游戏结果结构化类型**：
- `gameOutcome` 字段使用 `oneof` 类型，根据不同游戏返回对应的结构化数据
- 支持的类型：`diceOutcome`、`minesOutcome`、`blackjackOutcome`
- 每种游戏结果都有明确的字段定义，提供类型安全性

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

**令牌获取**: 必须通过Provider API的CreateSession接口获取

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

### 4. JWT Token刷新机制

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

### 6. 事件类型枚举

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
  EVENT_TYPE_BALANCE_UPDATE = 10;       // 余额更新通知
}
```

#### 事件类型说明

| 事件类型 | 值 | 描述 | 自动订阅 |
|---------|---|------|---------|
| EVENT_TYPE_BET_ACTIVITY_BATCH | 8 | 批量投注活动推送 | 是 |
| EVENT_TYPE_GAME_RESULT | 2 | 游戏结果事件 | 否 |
| EVENT_TYPE_BIG_WIN | 5 | 大额获胜通知 | 否 |
| EVENT_TYPE_JACKPOT | 6 | 累积奖池中奖 | 否 |
| EVENT_TYPE_LIVE_STATS | 9 | 实时统计更新 | 否 |
| EVENT_TYPE_BALANCE_UPDATE | 10 | 余额变化通知 | 是 |

**注意**：
- EVENT_TYPE_BET_ACTIVITY_BATCH：所有客户端连接后自动接收
- EVENT_TYPE_BALANCE_UPDATE：认证后自动推送，无需手动订阅

### 7. 消息类型

#### 1. 登录认证
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
- `client_seed`: 客户端种子 **（必填）**
  - 来源：由前端客户端生成的随机字符串
  - 用途：用于可证明公平机制，确保游戏结果的公正性
  - **格式要求**：
    - **必须提供**：不能为空或省略
    - **最小长度**：8个字符
    - **最大长度**：256个字符
    - **建议格式**：包含时间戳的随机字符串，如 "my-seed-" + Date.now()
    - **示例**：`"my-lucky-seed-123456"`, `"550e8400-e29b-41d4-a716-446655440000"`
  - **错误码**：`INVALID_CLIENT_SEED` - 当种子缺失或不符合要求时返回

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

**技术说明**：所有布尔类型字段（如 `isWin`）即使值为 `false` 也会在响应中返回。这是通过设置 `protojson.MarshalOptions.EmitUnpopulated = true` 实现的，确保客户端能正确接收所有字段。
```

**注意**：余额信息不再返回，由聚合器管理。

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

**配置格式说明**: 
- GameConfig 使用 `google.protobuf.Any` 类型包装配置
- 系统可以支持任意类型的游戏配置
- Dice 游戏：配置被包装为 `DiceGameConfig` 消息
- 其他游戏类型：可以动态添加新的配置类型
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
其他游戏（如 Mines、Blackjack、Keno）的配置结构类似，但会有各自特定的游戏参数。基本结构包含：
- 基本信息（id、gameId、gameName 等）
- 下注限制（betInfo 数组）
- RTP 配置
- 游戏特定参数

**Keno 游戏配置示例**:
```json
{
  "gameId": "inhousegame:keno",
  "config": {
    "id": 2000006,
    "gameId": "inhousegame:keno",
    "gameName": "Keno",
    "category": "instant",
    "status": "active",
    "description": "Classic lottery-style instant game with up to 10 number selections",
    "thumbnail": "/games/keno/thumbnail.png",
    "betInfo": [
      {
        "currency": "USD",
        "currencyType": "fiat",
        "defaultBet": 1.0,
        "minBet": 0.1,
        "maxBet": 1000
      }
    ],
    "minBet": 0.1,
    "maxBet": 1000,
    "rtp": 75.0,  // Keno通常RTP较低（70-80%）
    "features": ["provably_fair", "instant_play", "quick_pick"],
    "defaultRtp": "75%",
    "betRange": "0.1,1000",
    "maxRewardMultiplier": 100000,  // 最高10万倍（选10中10）
    "gameParameters": {
      "numberPool": {
        "min": 1,
        "max": 40    // 实际系统支持1-80，但当前限制为1-40
      },
      "spotRange": {
        "min": 1,     // 最少选1个数字
        "max": 10     // 最多选10个数字
      },
      "drawnNumbers": 10,  // 系统开出10个中奖号码
      "quickPickEnabled": true  // 支持快速选号
    }
  }
}
```




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

#### 6. 实时投注活动推送
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

#### 8. 获取余额
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

#### 9. 余额更新通知
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

### WebSocket 游戏消息格式

#### Mines 游戏 WebSocket 消息

**开始游戏（使用通用 PLACE_BET_REQUEST）**:
```json
{
  "id": "msg_123",
  "type": "PLACE_BET_REQUEST",
  "payload": {
    "game_type": "mines",
    "amount": "100.0",
    "currency": "USD",
    "game_params": {
      "mines": {
        "mines_count": 5,
        "grid_type": "5x5"  // 可选: "3x3", "5x5", "7x7"
      }
    },
    "client_seed": "player-chosen-seed"  // 必需：8-256字符
  }
}
```

**揭示格子请求（MINES_REVEAL_TILE）**:
```json
{
  "id": "msg_124",
  "type": "MINES_REVEAL_TILE",
  "payload": {
    "gameId": "inhousegame:mines:123456",
    "tileIndex": 12
  }
}
```

**提现请求（MINES_CASH_OUT）**:
```json
{
  "id": "msg_125",
  "type": "MINES_CASH_OUT",
  "payload": {
    "gameId": "inhousegame:mines:123456"
  }
}
```

**检查活跃游戏（MINES_CHECK_ACTIVE）**:
```json
{
  "id": "msg_126",
  "type": "MINES_CHECK_ACTIVE",
  "payload": {}
}
```

**恢复游戏（MINES_RESUME_GAME）**:
```json
{
  "id": "msg_127",
  "type": "MINES_RESUME_GAME",
  "payload": {
    "roundId": "inhousegame:mines:123456"
  }
}
```

**注意事项**:
1. **RoundID 格式**: 纯数字字符串（如 `"123456789012345678"`）- 使用 Sony Flake ID 生成器生成的唯一标识
2. **客户端种子**: 必须提供，8-256字符，无默认值
3. **单游戏限制**: 每个玩家同时只能有一个活跃的 Mines 游戏
4. **自动提现**: 5分钟无活动且有已揭示格子时自动提现
5. **余额更新**: 通过 WebSocket BALANCE_UPDATE 事件自动推送

#### Blackjack 游戏 WebSocket 消息

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

#### Keno 游戏 WebSocket 消息 ✅ 已实现

**开始游戏（使用通用 PLACE_BET_REQUEST）**:
```json
{
  "id": "msg_300",
  "type": "PLACE_BET_REQUEST",
  "payload": {
    "amount": "100.0",
    "gameParams": {
      "keno": {
        "selectedNumbers": [1, 5, 10, 15, 20]  // 选择的数字（1-80），必须包含1-10个不重复的号码
      }
    },
    "clientSeed": "player-chosen-seed-789"  // 必需：8-256字符
  }
}
```

> **注意**: 快速选号（Quick Pick）功能已移至客户端处理，服务端只负责验证号码的合法性。客户端可自行实现随机选号功能，然后通过 `selectedNumbers` 字段发送选中的号码。

**响应消息（PLACE_BET_RESPONSE）**:
```json
{
  "id": "msg_302",
  "type": "PLACE_BET_RESPONSE",
  "payload": {
    "betId": "123456789012345678",
    "gameResult": {
      "gameId": "inhousegame:keno",
      "betAmount": "100.0",
      "winAmount": "200.0",
      "isWin": true,
      "gameOutcome": {
        "kenoOutcome": {
          "selectedNumbers": [1, 5, 10, 15, 20],  // 玩家选择的数字
          "drawnNumbers": [1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 27, 29, 31, 33, 35, 37, 39],  // 系统开出的20个数字
          "matchedNumbers": [1, 5, 15],  // 匹配的数字
          "matchCount": 3,  // 匹配数量
          "spotsCount": 5,  // 选择的数字数量
          "multiplier": "2.0"  // 赔率
        }
      },
      "multiplier": "2.0",
      "timestamp": 1640995200000
    },
    "balance": "1100.0"
  }
}
```

**注意事项**:
1. **即时游戏**: Keno是即时游戏，一次下注立即完成，无需会话管理
2. **数字范围**: 选择的数字必须在1-80范围内
3. **选择数量**: 可选择1-10个数字
4. **快速选号**: 支持系统自动随机选择指定数量的数字
5. **赔率计算**: 根据选择数量和匹配数量查询赔率表
6. **RoundID格式**: 纯数字字符串，使用Sony Flake ID生成器

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

### 说明
为了避免浮点数精度损失问题，所有涉及金额的字段使用 `string` 类型。

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


## 相关文档
- [详细设计](./detailed-design-zh.md) - 架构和设计原则
- [序列图](./sequence-diagrams-zh.md) - 可视化流程展示
- [集成指南](others/integration-guide-zh.md) - 分步集成说明