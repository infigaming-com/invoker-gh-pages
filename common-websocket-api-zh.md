# WebSocket API 通用接口文档

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

获取游戏配置信息，包括投注限额、RTP、游戏参数等。配置支持运营商级别覆盖。

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

#### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `gameId` | string | 游戏唯一标识，格式：`inhousegame:gametype` |
| `status` | string | 游戏状态：`active` / `inactive` |
| `rtp` | number | 返还率（百分比），如 97.0 表示 97% |
| `betInfo` | array | 各货币的投注限额配置 |
| `gameParameters` | object | 游戏特定参数 |
| `payoutTables` | object | 赔率表（仅表驱动游戏有此字段） |

**betInfo 字段结构**：
```json
{
  "currency": "USD",
  "currencyType": "fiat",
  "defaultBet": 1,
  "minBet": 0.1,
  "maxBet": 10000,
  "maxProfit": 100000
}
```

**响应消息**（Dice 游戏 - 公式驱动）：
```json
{
  "i": "msg_002",
  "t": "GET_GAME_CONFIG",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:dice",
        "config": {
          "gameId": "inhousegame:dice",
          "status": "active",
          "rtp": 97,
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 1,
              "minBet": 0.1,
              "maxBet": 10000,
              "maxProfit": 100000
            }
          ],
          "gameParameters": {
            "rollNumberMin": 4,
            "rollNumberMax": 96,
            "defaultTarget": 50
          }
        }
      }
    ]
  }
}
```

**响应消息**（Keno 游戏 - 表驱动）：
```json
{
  "i": "msg_002",
  "t": "GET_GAME_CONFIG",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:keno",
        "config": {
          "gameId": "inhousegame:keno",
          "status": "active",
          "rtp": 97,
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 1,
              "minBet": 0.1,
              "maxBet": 1000,
              "maxProfit": 100000
            }
          ],
          "gameParameters": {
            "defaultDifficulty": "classic",
            "minSpots": 1,
            "maxSpots": 10
          },
          "payoutTables": {
            "classic": {
              "payouts": [
                {
                  "spots": 1,
                  "entries": [
                    {"hits": 0, "payout": 0},
                    {"hits": 1, "payout": 3.96}
                  ]
                },
                {
                  "spots": 2,
                  "entries": [
                    {"hits": 0, "payout": 0},
                    {"hits": 2, "payout": 3.8}
                  ]
                }
              ]
            },
            "low": { "payouts": [...] },
            "medium": { "payouts": [...] },
            "high": { "payouts": [...] }
          }
        }
      }
    ]
  }
}
```

**payoutTables 结构说明**（表驱动游戏）：
- 顶级为难度/风险等级的 map（如 `classic`、`low`、`medium`、`high`）
- 每个难度包含 `payouts` 数组，按选择数量（spots）分组
- 每个 payout 条目包含：
  - `spots`: 玩家选择的数字数量
  - `entries`: 命中数量与赔率的映射

**响应消息**（Plinko 游戏 - 表驱动）：
```json
{
  "i": "msg_003",
  "t": "GET_GAME_CONFIG",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:plinko",
        "config": {
          "gameId": "inhousegame:plinko",
          "status": "active",
          "rtp": 97,
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 0.2,
              "minBet": 0.1,
              "maxBet": 400,
              "maxProfit": 20000
            }
          ],
          "gameParameters": {
            "defaultDifficulty": "medium",
            "defaultRows": 14
          },
          "payoutTables": {
            "low": {
              "rows": {
                "8": { "multipliers": [5.6, 2.1, 1.1, 1, 0.5, 1, 1.1, 2.1, 5.6] },
                "10": { "multipliers": [8.9, 3, 1.4, 1.1, 1, 0.5, 1, 1.1, 1.4, 3, 8.9] }
              }
            },
            "medium": {
              "rows": {
                "8": { "multipliers": [13, 3, 1.3, 0.7, 0.4, 0.7, 1.3, 3, 13] }
              }
            },
            "high": {
              "rows": {
                "8": { "multipliers": [29, 4, 1.5, 0.3, 0.2, 0.3, 1.5, 4, 29] }
              }
            }
          }
        }
      }
    ]
  }
}
```

**Plinko payoutTables 结构**：
- 第一层：难度等级（`low`、`medium`、`high`）
- 第二层 `rows`：按行数分类（`8`、`10`、`12`、`14`、`16`）
- `multipliers` 数组：槽位倍率，长度 = 行数 + 1

**响应消息**（Dragon Tiger 游戏 - 公式驱动）：
```json
{
  "i": "msg_004",
  "t": "GET_GAME_CONFIG",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:dragontiger",
        "config": {
          "gameId": "inhousegame:dragontiger",
          "status": "active",
          "rtp": 98.5,
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 1,
              "minBet": 0.1,
              "maxBet": 1000,
              "maxProfit": 50000
            }
          ],
          "gameParameters": {
            "commissionRate": 0.05,
            "dragonOdds": 1.0,
            "tigerOdds": 1.0,
            "tieOdds": 11.0,
            "tieRefundRate": 0.5
          }
        }
      }
    ]
  }
}
```

**Dragon Tiger gameParameters 说明**：
- `commissionRate`: 佣金费率（5%）
- `dragonOdds`: 龙方赔率（1:1）
- `tigerOdds`: 虎方赔率（1:1）
- `tieOdds`: 和局赔率（1:11）
- `tieRefundRate`: 平局退款比例（50%）

### 2.3 GET_BALANCE - 获取余额

获取当前玩家余额。系统自动使用会话创建时的币种，无需在请求中指定。

**请求消息**：
```json
{
  "i": "msg_003",
  "t": "GET_BALANCE",
  "p": {}  // 无需currency字段，自动使用会话币种
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
    "eventTypes": ["BET_ACTIVITY_BATCH"],
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
    "eventTypes": ["BET_ACTIVITY_BATCH"],
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
    "eventTypes": ["BET_ACTIVITY_BATCH"],  // 可选，不提供则取消所有订阅
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

> **注意**: 如需获取投注活动历史，请使用 RESTful API `POST /v1/bets/activities`，该接口不需要认证。详见 [API 文档](https://storage.googleapis.com/speedix-invoker-api-docs/index.html)。

## 3. 事件推送

### 3.1 INITIALIZATION_COMPLETE - 初始化完成

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

### 3.2 BET_ACTIVITY_BATCH - 投注活动批量广播

实时批量广播其他玩家的投注活动（需要先订阅）。系统会对投注活动进行采样和批量处理，以优化网络传输。

**推送消息**：
```json
{
  "i": "server_msg_003",
  "t": "BET_ACTIVITY_BATCH",
  "p": {
    "activities": [
      {
        "maskedPlayerId": "pla***456",
        "maskedUsername": "Pla***456",
        "gameId": "inhousegame:dice",
        "betAmount": "100.00000000",
        "potentialWin": "200.00000000",
        "multiplier": "2.00000000",
        "isWin": true,
        "winAmount": "200.00000000",
        "currency": "USD",
        "countryCode": "US",
        "timestamp": 1704067220
      }
    ]
  }
}
```

**字段说明**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `maskedPlayerId` | string | 脱敏后的玩家 ID（**已废弃**，将在未来版本移除，请使用 `maskedUsername`） |
| `maskedUsername` | string | 脱敏后的用户名（推荐使用） |
| `gameId` | string | 游戏 ID |
| `betAmount` | string | 投注金额（8位小数精度） |
| `potentialWin` | string | 潜在赢取金额 |
| `multiplier` | string | 倍率 |
| `isWin` | bool | 是否赢了（可选，游戏结束时才有） |
| `winAmount` | string | 实际赢取金额（仅当 isWin=true 时有值） |
| `currency` | string | 货币类型 |
| `countryCode` | string | 国家代码 |
| `timestamp` | int64 | Unix 秒时间戳 |

**注意事项**：
- 投注活动会根据金额大小进行采样，大额投注和大额赢取更容易被广播
- 消息以批量形式推送，每批最多包含 20 条活动记录
- 用户名使用脱敏处理，保护玩家隐私（如 "Player456" → "Pla***456"）

## 4. 心跳机制

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

## 5. 错误处理

### 5.1 错误响应格式

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

### 5.2 常见错误码

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

## 6. 金额格式说明

### 6.1 说明
为了避免浮点数精度损失问题，所有涉及金额的字段使用 `string` 类型。

### 6.2 金额字段规则

#### 格式要求
- 所有金额必须以字符串形式传输
- 保留 8 位小数，如：`"123.45678901"`
- 不使用科学计数法
- 最小值：`"0.00000001"`
- 最大值：取决于具体业务限制

#### 受影响的字段

**WebSocket API**:
- `amount`: 下注金额
- `balance`: 余额
- `bet_amount`: 下注金额
- `win_amount`: 获胜金额
- `multiplier`: 倍数
- `payout`: 支付金额

**统计数据字段**:
- `total_volume`: 总交易量
- `total_winnings`: 总获胜金额
- `biggest_win`: 最大获胜金额
- `avg_bet_size`: 平均下注金额

#### 示例

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
    "roundId": "bet_123",
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

## 7. 注意事项

### 7.1 连接管理
- WebSocket连接会自动维护心跳，无需手动发送ping/pong
- 连接断开后需要重新认证
- 建议实现断线重连机制

### 7.2 消息ID管理
- 客户端应为每个请求生成唯一的消息ID
- 通过消息ID关联请求和响应
- 建议使用UUID或时间戳+随机数

### 7.3 数值精度
- 所有金额相关字段使用字符串类型，避免浮点数精度问题
- 示例："10.50" 而不是 10.5

### 7.4 时间戳格式
- 所有时间戳使用Unix毫秒时间戳（int64）
- 示例：1704067200000

### 7.5 游戏ID格式
- 游戏ID格式为："inhousegame:游戏类型"
- 示例："inhousegame:dice"、"inhousegame:mines"、"inhousegame:keno"

### 7.6 客户端种子管理

客户端种子通过专门的 Seed Service API 管理，不在游戏请求中直接传递：

1. **设置客户端种子**：使用 `UseNewSeeds` API 为特定游戏设置新的客户端种子
2. **生成安全种子**：使用 `GenerateClientSeed` API 生成安全的随机种子
3. **获取种子信息**：使用 `GetGameSeedInfo` API 查看当前种子信息
4. **自动获取**：游戏时系统自动使用已设置的客户端种子

注意事项：
- 必须在游戏开始前设置客户端种子（8-256字符）
- 游戏进行中无法更改种子
- 每个游戏独立管理种子
- 用于可证明公平机制

### 7.7 试玩模式
- 所有游戏支持试玩模式
- 将 `amount` 设为空字符串 `""` 或 `"0"` 即可激活
- 试玩模式下不扣除余额，但游戏逻辑完全相同
- 适合新用户了解游戏规则和体验游戏

## 8. 相关文档

### 游戏专属 API 文档
- [Dice 游戏 WebSocket API](./dice-websocket-api-zh.md)
- [Mines 游戏 WebSocket API](./mines-websocket-api-zh.md)
- [Keno 游戏 WebSocket API](./keno-websocket-api-zh.md)
- [Plinko 游戏 WebSocket API](./plinko-websocket-api-zh.md)
- [HiLo 游戏 WebSocket API](./hilo-websocket-api-zh.md)
- [Chicken Road 游戏 WebSocket API](./chickenroad-websocket-api-zh.md)
- [Dragon Tiger 游戏 WebSocket API](./dragontiger-websocket-api-zh.md)

### 其他文档
- [序列图](./sequence-diagrams-zh.md) - 可视化流程展示