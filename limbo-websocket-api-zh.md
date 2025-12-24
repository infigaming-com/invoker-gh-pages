# Limbo WebSocket API 参考文档

## 状态：✅ 已实现

## 1. 概述

Limbo 游戏使用 WebSocket 协议进行实时通信，客户端通过 `PLACE_BET` 消息进行投注，服务端返回游戏结果。

### 1.1 基本信息

- **协议**：WebSocket (wss://)
- **端口**：8001
- **认证方式**：JWT Token（Bearer）
- **消息格式**：Protobuf (JSON 编码)
- **Game ID**：`inhousegame:limbo`

### 1.2 通用消息结构

所有 WebSocket 消息遵循统一格式：

```json
{
  "i": "message_id",      // 消息 ID（客户端生成）
  "t": "MESSAGE_TYPE",    // 消息类型
  "p": {                  // 消息载荷（Protobuf Any 类型）
    "@type": "type.googleapis.com/api.game.v1.MessageName",
    "field1": "value1",
    ...
  }
}
```

**字段说明**：
- `i` (id): 消息唯一标识符，客户端生成，用于关联请求和响应
- `t` (type): 消息类型字符串
- `p` (payload): 消息载荷，使用 Protobuf Any 类型封装

## 2. 认证流程

### 2.1 LOGIN

客户端连接后必须先发送 LOGIN 消息进行认证。

**请求**：

```json
{
  "i": "msg_001",
  "t": "LOGIN",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.LoginRequest",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**响应**：

```json
{
  "i": "msg_001",
  "t": "LOGIN_RESPONSE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.LoginResponse",
    "success": true,
    "userId": "player_123",
    "gameId": "inhousegame:limbo"
  }
}
```

## 3. 核心游戏接口

### 3.1 PLACE_BET - 下注

玩家设定目标倍率并下注，系统立即生成结果并返回。

**请求消息**：

```json
{
  "i": "bet_001",
  "t": "PLACE_BET",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.PlaceBetRequest",
    "amount": "10.00",
    "gameParams": {
      "limbo": {
        "targetMultiplier": "2.00"
      }
    }
  }
}
```

**请求参数**：

| 字段 | 类型 | 必填 | 说明 | 范围 |
|------|------|------|------|------|
| amount | string | 是 | 下注金额 | 0.000100 - 500,000 USD |
| gameParams.limbo.targetMultiplier | string | 是 | 目标倍率 | 1.01 - 10.00 |

**响应消息**：

```json
{
  "i": "bet_001",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.PlaceBetResponse",
    "roundId": "1234567890123456789",
    "gameResult": {
      "betAmount": "10.00",
      "winAmount": "20.00",
      "isWin": true,
      "multiplier": "2.00",
      "timestamp": 1704067200000,
      "limboOutcome": {
        "resultMultiplier": "2.35",
        "targetMultiplier": "2.00"
      }
    },
    "balance": "1020.00"
  }
}
```

**响应字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| roundId | string | 回合 ID（用于查询和验证） |
| gameResult | object | 游戏结果对象 |
| gameResult.betAmount | string | 下注金额 |
| gameResult.winAmount | string | 获胜金额（失败时为 "0.00"） |
| gameResult.isWin | boolean | 是否获胜 |
| gameResult.multiplier | string | 实际倍率（= targetMultiplier） |
| gameResult.timestamp | int64 | Unix 毫秒时间戳 |
| gameResult.limboOutcome.resultMultiplier | string | 随机生成的结果倍率 |
| gameResult.limboOutcome.targetMultiplier | string | 玩家设定的目标倍率 |
| balance | string | 更新后的余额 |

**判定逻辑**：
```
如果 resultMultiplier >= targetMultiplier：
  获胜，派彩 = betAmount × targetMultiplier
否则：
  失败，损失全部下注金额
```

### 3.2 游戏结果示例

**示例 1：获胜场景**

```json
{
  "roundId": "1234567890123456789",
  "gameResult": {
    "betAmount": "100.00",
    "winAmount": "1000.00",
    "isWin": true,
    "multiplier": "10.00",
    "timestamp": 1704067200000,
    "limboOutcome": {
      "resultMultiplier": "15.67",
      "targetMultiplier": "10.00"
    }
  },
  "balance": "1900.00"
}
```

**说明**：
- 下注 100，目标倍率 10x
- 结果倍率 15.67 ≥ 10，获胜
- 派彩 = 100 × 10 = 1000
- 余额从 1000 变为 1900（-100 + 1000）

**示例 2：失败场景**

```json
{
  "roundId": "1234567890123456790",
  "gameResult": {
    "betAmount": "50.00",
    "winAmount": "0.00",
    "isWin": false,
    "multiplier": "5.00",
    "timestamp": 1704067260000,
    "limboOutcome": {
      "resultMultiplier": "3.42",
      "targetMultiplier": "5.00"
    }
  },
  "balance": "850.00"
}
```

**说明**：
- 下注 50，目标倍率 5x
- 结果倍率 3.42 < 5，失败
- 损失全部下注金额 50
- 余额从 900 变为 850

**示例 3：高倍率挑战**

```json
{
  "roundId": "1234567890123456791",
  "gameResult": {
    "betAmount": "1.00",
    "winAmount": "10000.00",
    "isWin": true,
    "multiplier": "10000.00",
    "timestamp": 1704067320000,
    "limboOutcome": {
      "resultMultiplier": "12345.67",
      "targetMultiplier": "10000.00"
    }
  },
  "balance": "10849.00"
}
```

## 4. 辅助接口

### 4.1 GET_BALANCE - 获取余额

**请求**：

```json
{
  "i": "balance_001",
  "t": "GET_BALANCE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.GetBalanceRequest"
  }
}
```

**响应**：

```json
{
  "i": "balance_001",
  "t": "GET_BALANCE_RESPONSE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.GetBalanceResponse",
    "balance": "1000.00",
    "currency": "USD"
  }
}
```

### 4.2 GET_GAME_CONFIG - 获取游戏配置

**请求**：

```json
{
  "i": "config_001",
  "t": "GET_GAME_CONFIG",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.GetGameConfigRequest",
    "gameId": "inhousegame:limbo"
  }
}
```

**响应**：

```json
{
  "i": "config_001",
  "t": "GET_GAME_CONFIG_RESPONSE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.GetGameConfigResponse",
    "configs": [
      {
        "gameId": "inhousegame:limbo",
        "config": {
          "@type": "type.googleapis.com/api.game.v1.LimboGameConfig",
          "id": 2000007,
          "gameName": "Limbo",
          "gameId": "inhousegame:limbo",
          "category": "instant",
          "status": "active",
          "description": "Multiplier prediction game",
          "thumbnail": "/games/limbo/thumbnail.png",
          "defaultRTP": "99%",
          "features": ["provably_fair", "instant_play", "turbo_mode"],
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 10.0,
              "minBet": 0.0001,
              "maxBet": 500000.0,
              "maxProfit": 5000000.0
            }
          ],
          "gameParameters": {
            "minMultiplier": "1.01",
            "maxMultiplier": "10.00",
            "defaultMultiplier": "2.00"
          },
          "commissionRate": "1%",
          "maxRewardMultiplier": 10.0
        }
      }
    ]
  }
}
```

## 5. 事件订阅（可选）

### 5.1 SUBSCRIBE - 订阅事件

订阅投注活动和余额更新事件。

**请求**：

```json
{
  "i": "sub_001",
  "t": "SUBSCRIBE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.SubscribeRequest",
    "eventTypes": ["BET_ACTIVITY"]
  }
}
```

**响应**：

```json
{
  "i": "sub_001",
  "t": "SUBSCRIBE_RESPONSE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.SubscribeResponse",
    "subscriptionId": "sub_12345",
    "eventTypes": ["BET_ACTIVITY"],
    "success": true,
    "message": "Successfully subscribed"
  }
}
```

## 6. 错误处理

### 6.1 错误消息格式

```json
{
  "i": "bet_001",
  "t": "ERROR",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.ErrorMessage",
    "code": "INSUFFICIENT_BALANCE",
    "message": "余额不足",
    "details": "当前余额: 5.00, 所需金额: 10.00",
    "requestId": "bet_001"
  }
}
```

### 6.2 常见错误代码

| 错误代码 | 说明 | 解决方案 |
|---------|------|----------|
| UNAUTHORIZED | 未认证 | 先发送 LOGIN 消息 |
| INVALID_TARGET_MULTIPLIER | 无效的目标倍率 | 倍率范围：1.01 - 10.00 |
| INVALID_AMOUNT | 无效的下注金额 | 金额范围：0.000100 - 500,000 |
| INSUFFICIENT_BALANCE | 余额不足 | 充值或减少下注金额 |
| PAYOUT_LIMIT_EXCEEDED | 超过赔付上限 | 减少下注金额或目标倍率 |
| GAME_IN_PROGRESS | 游戏进行中 | 等待当前游戏完成 |
| INVALID_GAME_PARAMS | 游戏参数无效 | 检查参数格式和范围 |

## 7. 可证明公平验证

### 7.1 获取种子信息（暂未实现）

未来将提供种子管理接口：

```json
{
  "i": "seed_001",
  "t": "GET_SEED_INFO",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.GetSeedInfoRequest",
    "gameId": "inhousegame:limbo"
  }
}
```

**响应**：

```json
{
  "i": "seed_001",
  "t": "GET_SEED_INFO_RESPONSE",
  "p": {
    "@type": "type.googleapis.com/api.game.v1.GetSeedInfoResponse",
    "clientSeed": "my_client_seed_123",
    "hashedServerSeed": "a1b2c3d4e5f6...",
    "nonce": 42
  }
}
```

### 7.2 验证算法

**第一步：生成哈希**

```javascript
const crypto = require('crypto');

function generateHash(clientSeed, serverSeed, nonce) {
  const seedStr = `${clientSeed}:${serverSeed}:${nonce}`;
  const hash = crypto.createHash('sha256').update(seedStr).digest('hex');
  return hash;
}
```

**第二步：计算倍率**

```javascript
function calculateMultiplier(hash) {
  const hashFirst8 = hash.substring(0, 8);
  const hashDecimal = parseInt(hashFirst8, 16);
  const max32Bit = 0xFFFFFFFF;
  const r = hashDecimal / max32Bit;  // [0, 1)

  if (r >= 0.99) {
    // 极限情况保护
    return 10.00;
  }

  const multiplier = 0.99 / (1 - r);
  return Math.min(multiplier, 10.00);
}
```

**第三步：验证结果**

```javascript
function verifyResult(clientSeed, serverSeed, nonce, expectedMultiplier) {
  const hash = generateHash(clientSeed, serverSeed, nonce);
  const multiplier = calculateMultiplier(hash);

  // 允许小数精度误差（±0.01）
  const diff = Math.abs(multiplier - expectedMultiplier);
  return diff < 0.01;
}

// 示例
const isValid = verifyResult(
  "my_client_seed",
  "server_seed_revealed",
  42,
  15.67
);
console.log(isValid ? "验证通过" : "验证失败");
```

## 8. 完整交互流程示例

### 8.1 标准游戏流程

```javascript
// 1. 建立 WebSocket 连接
const ws = new WebSocket('wss://game.example.com:8001');

// 2. 发送 LOGIN
ws.send(JSON.stringify({
  i: "msg_001",
  t: "LOGIN",
  p: {
    "@type": "type.googleapis.com/api.game.v1.LoginRequest",
    token: "your_jwt_token"
  }
}));

// 3. 接收 LOGIN_RESPONSE
// ... 等待响应 ...

// 4. 发送 PLACE_BET
ws.send(JSON.stringify({
  i: "bet_001",
  t: "PLACE_BET",
  p: {
    "@type": "type.googleapis.com/api.game.v1.PlaceBetRequest",
    amount: "100.00",
    gameParams: {
      limbo: {
        targetMultiplier: "10.00"
      }
    }
  }
}));

// 5. 接收 PLACE_BET_RESPONSE
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.t === "PLACE_BET_RESPONSE") {
    const result = msg.p.gameResult;
    console.log(`结果倍率: ${result.limboOutcome.resultMultiplier}`);
    console.log(`目标倍率: ${result.limboOutcome.targetMultiplier}`);
    console.log(`是否获胜: ${result.isWin}`);
    console.log(`派彩金额: ${result.winAmount}`);
    console.log(`新余额: ${msg.p.balance}`);
  }
};
```

### 8.2 极速模式（客户端控制）

极速模式通过客户端跳过动画实现，服务端行为不变。客户端可以：

1. 在 UI 上提供"极速模式"开关
2. 开启后，接收到 `PLACE_BET_RESPONSE` 立即显示结果，跳过火箭动画
3. 关闭后，播放 0.8-1.2 秒的火箭攀升动画

服务端可以在 `PlaceBetRequest` 中添加 `turbo` 标志（仅用于日志记录）：

```json
{
  "amount": "10.00",
  "gameParams": {
    "limbo": {
      "targetMultiplier": "2.00"
    }
  },
  "turbo": true  // 可选，仅供服务端统计
}
```

## 9. 最佳实践

### 9.1 客户端种子管理

- 使用安全的随机数生成器（如 `crypto.getRandomValues()`）
- 长度：8-256 字符
- 可包含字母、数字、特殊字符
- 建议定期更换

### 9.2 网络异常处理

- 实现消息重试机制（使用相同的 `i` 保证幂等）
- 连接断开后重新 LOGIN
- 超时时间建议：5 秒（普通）、2 秒（极速）

### 9.3 UI/UX 建议

- 显示实时胜率：`winChance = 0.99 / targetMultiplier`
- 显示潜在派彩：`potentialWin = betAmount × targetMultiplier`
- 历史记录：显示最近 25 局的结果倍率
- 倍率输入：使用滑杆（对数刻度）+ 输入框组合

## 10. 相关文档

- [Limbo 详细设计文档](./limbo-detailed-design-zh.md)
- [WebSocket API 索引](./websocket-api-reference-zh.md)
- [通用 WebSocket API](./common-websocket-api-zh.md)
- [可证明公平机制](./provably-fair-zh.md)