# Centrifugo RPC 通用接口文档

## 1. 概述

Invoker Server 使用 Centrifugo 作为实时通信层，通过 RPC 调用处理游戏下注、状态同步等功能。

### 1.1 连接信息

| 环境 | 连接地址 |
|------|----------|
| 本地开发 | `ws://localhost:8001/connection/websocket` |
| 开发环境 | `wss://centrifugo.dev.hicasino.xyz/connection/websocket` |

### 1.2 客户端库

使用 [Centrifuge JavaScript SDK](https://github.com/centrifugal/centrifuge-js)（v5.5.2，兼容服务端 Centrifugo v6）：

```html
<!-- JSON 协议（默认） -->
<script src="https://cdn.jsdelivr.net/npm/centrifuge@5.5.2/dist/centrifuge.min.js"></script>

<!-- Protobuf 协议（可选，更高效） -->
<script src="https://cdn.jsdelivr.net/npm/centrifuge@5.5.2/dist/centrifuge.protobuf.min.js"></script>
```

### 1.3 协议支持

- **JSON**（默认）：易于调试，适合开发环境
- **Protobuf**：二进制格式，更高效，适合生产环境

## 2. 连接和认证

### 2.1 认证流程

系统采用 Connect Proxy 模式，认证在连接阶段完成：

1. **创建会话**：通过 Provider API 创建游戏会话，获取 JWT token
2. **连接 Centrifugo**：在连接时传递 token 进行认证
3. **调用 RPC**：认证成功后即可调用游戏 RPC

### 2.2 连接代码示例

```javascript
// 1. 通过 Provider API 创建会话获取 token
const response = await fetch('/sessions', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-API-KEY': apiKey,
        'X-SIGNATURE': signature
    },
    body: JSON.stringify(sessionData)
});
const result = await response.json();
const token = result.data.launch_options.game_url;

// 2. 使用 token 连接 Centrifugo
const centrifuge = new Centrifuge(
    'wss://centrifugo.dev.hicasino.xyz/connection/websocket',
    { data: { token: token } }
);

// 3. 监听连接事件
centrifuge.on('connected', (ctx) => {
    console.log('已连接，Client ID:', ctx.client);
});

centrifuge.on('disconnected', (ctx) => {
    console.log('断开连接:', ctx.reason);
});

centrifuge.on('error', (ctx) => {
    console.error('连接错误:', ctx);
});

// 4. 建立连接
centrifuge.connect();
```

### 2.3 Protobuf 协议连接

```javascript
const centrifuge = new Centrifuge(
    'wss://centrifugo.dev.hicasino.xyz/connection/websocket',
    {
        protocol: 'protobuf',
        getData: function() {
            const jsonStr = JSON.stringify({ token: token });
            return Promise.resolve(new TextEncoder().encode(jsonStr));
        }
    }
);
```

## 3. RPC 调用

### 3.1 调用格式

```javascript
const result = await centrifuge.rpc(method, data);
```

- **method**：RPC 方法名，格式为 `{namespace}.{action}`
- **data**：请求参数对象

### 3.2 方法命名规范

| 命名空间 | 说明 | 示例 |
|----------|------|------|
| `common` | 通用操作 | `common.getBalance`、`common.getConfig` |
| `{gameName}` | 游戏操作 | `dice.placeBet`、`mines.reveal` |

### 3.3 响应处理

```javascript
try {
    const result = await centrifuge.rpc('common.getBalance', {});
    // result.data 包含响应数据
    console.log('余额:', result.data.balance);
} catch (error) {
    // 错误处理
    console.error('RPC 错误:', error.message);
}
```

### 3.4 Protobuf 响应解码

```javascript
function parseRpcResponse(result) {
    let data = result.data;
    if (data instanceof Uint8Array) {
        const jsonStr = new TextDecoder().decode(data);
        data = JSON.parse(jsonStr);
    }
    return data;
}
```

## 4. 通用接口

### 4.1 common.getBalance - 获取余额

获取当前玩家余额。

**请求**：
```javascript
const result = await centrifuge.rpc('common.getBalance', {});
```

**响应**：
```json
{
    "balance": "1000.50000000",
    "currency": "USDT"
}
```

### 4.2 common.getConfig - 获取游戏配置

获取游戏配置信息，包括投注限额、RTP、游戏参数等。

**请求**：
```javascript
const result = await centrifuge.rpc('common.getConfig', {
    gameId: 'inhousegame:dice'
});
```

**响应**（Dice 游戏示例）：
```json
{
    "gameId": "inhousegame:dice",
    "config": {
        "gameId": "inhousegame:dice",
        "status": "active",
        "rtp": 99,
        "betInfo": [
            {
                "currency": "USDT",
                "currencyType": "crypto",
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
```

**betInfo 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `currency` | string | 货币代码 |
| `currencyType` | string | 货币类型：`fiat` / `crypto` |
| `defaultBet` | number | 默认投注金额 |
| `minBet` | number | 最小投注金额 |
| `maxBet` | number | 最大投注金额 |
| `maxProfit` | number | 最大利润限制 |

## 5. 频道订阅

### 5.1 公开频道

公开频道无需认证即可订阅：

| 频道 | 说明 |
|------|------|
| `live_bets:*` | 实时投注活动广播 |
| `game:crash` | Crash 游戏实时事件 |

**订阅示例**：
```javascript
const subscription = centrifuge.newSubscription('live_bets:all');

subscription.on('publication', (ctx) => {
    const activity = ctx.data;
    console.log('投注活动:', activity);
});

subscription.subscribe();
```

### 5.2 私有频道

私有频道需要认证，在 Subscribe Proxy 中验证权限：

| 频道格式 | 说明 |
|----------|------|
| `user:{playerID}` | 用户专属消息 |

### 5.3 投注活动数据格式

```json
{
    "activities": [
        {
            "maskedPlayerId": "pla***456",
            "maskedUsername": "Pla***456",
            "gameId": "inhousegame:dice",
            "betAmount": "100.00000000",
            "multiplier": "2.00000000",
            "isWin": true,
            "winAmount": "200.00000000",
            "currency": "USDT",
            "countryCode": "VN",
            "timestamp": 1704067220
        }
    ]
}
```

## 6. 错误码

当 RPC 调用失败时，会抛出包含错误码的异常。

### 6.1 错误处理示例

```javascript
try {
    const result = await centrifuge.rpc('dice.placeBet', data);
} catch (error) {
    // error.message 包含错误信息
    // 可能的格式: "100: 余额不足"
    console.error('错误:', error.message);
}
```

### 6.2 错误码列表

| 错误码 | 名称 | 说明 | 处理建议 |
|--------|------|------|----------|
| 2 | INVALID_PARAMS | 参数无效 | 检查请求参数格式 |
| 3 | UNAUTHORIZED | 未授权 | 重新连接并认证 |
| 4 | INTERNAL | 内部错误 | 稍后重试 |
| 100 | INSUFFICIENT_BALANCE | 余额不足 | 提示用户充值 |
| 101 | INVALID_BET_AMOUNT | 投注金额无效 | 检查金额范围 |
| 102 | BET_LIMIT_EXCEEDED | 超出投注限额 | 调整投注金额 |
| 200 | GAME_NOT_FOUND | 游戏不存在 | 检查游戏 ID |
| 201 | GAME_NOT_STARTED | 游戏未开始 | 等待游戏开始 |
| 202 | GAME_ALREADY_STARTED | 游戏已开始 | 等待当前游戏结束 |
| 203 | GAME_ALREADY_FINISHED | 游戏已结束 | 开始新游戏 |
| 204 | INVALID_GAME_STATE | 游戏状态无效 | 检查游戏状态 |
| 205 | NO_ACTIVE_GAME | 没有活跃游戏 | 先开始游戏 |
| 300 | OPERATION_NOT_ALLOWED | 操作不允许 | 检查当前状态 |
| 301 | CANNOT_CASH_OUT | 无法提现 | 检查游戏状态 |
| 302 | INVALID_ACTION | 无效操作 | 检查操作类型 |
| 400 | CRASH_INVALID_PHASE | Crash 阶段无效 | 等待正确阶段 |
| 401 | CRASH_BET_NOT_FOUND | Crash 投注不存在 | 检查投注 ID |
| 402 | CRASH_ALREADY_CASHED_OUT | Crash 已兑现 | - |

## 7. 金额格式说明

为避免浮点数精度问题，所有金额使用字符串类型。

### 7.1 格式要求

- 使用字符串类型传输
- 保留 8 位小数：`"123.45678901"`
- 不使用科学计数法
- 最小值：`"0.00000001"`

### 7.2 涉及字段

- `amount` - 投注金额
- `balance` - 余额
- `betAmount` - 投注金额
- `winAmount` - 获胜金额
- `multiplier` - 倍数
- `payout` - 支付金额

### 7.3 试玩模式

将 `amount` 设为 `"0"` 或空字符串即可激活试玩模式：
- 不扣除余额
- 游戏逻辑完全相同
- 适合了解游戏规则

## 8. 连接管理

### 8.1 心跳机制

Centrifugo 自动管理心跳，无需手动发送。客户端库会自动：
- 发送心跳保持连接
- 检测连接状态
- 自动重连（可配置）

### 8.2 断线重连

```javascript
const centrifuge = new Centrifuge(url, {
    data: { token: token },
    // 自动重连配置
    minReconnectDelay: 1000,    // 最小重连延迟 (ms)
    maxReconnectDelay: 20000    // 最大重连延迟 (ms)
});
```

### 8.3 断开连接

```javascript
centrifuge.disconnect();
```

## 9. 相关文档

### 游戏专属 API 文档

- [Dice 游戏 API](./dice-websocket-api-zh.md)
- [Mines 游戏 API](./mines-websocket-api-zh.md)
- [Keno 游戏 API](./keno-websocket-api-zh.md)
- [Plinko 游戏 API](./plinko-websocket-api-zh.md)
- [HiLo 游戏 API](./hilo-websocket-api-zh.md)
- [Chicken Road 游戏 API](./chickenroad-websocket-api-zh.md)
- [Dragon Tiger 游戏 API](./dragontiger-websocket-api-zh.md)
- [Crash 游戏 API](./crash-websocket-api-zh.md)

### 其他文档

- [WebSocket API 索引](./websocket-api-reference-zh.md)
- [系统架构](./architecture-zh.md)
- [RESTful API 文档](https://storage.googleapis.com/speedix-invoker-api-docs/index.html)
