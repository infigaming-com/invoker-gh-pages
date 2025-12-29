# Centrifugo RPC API 文档索引

## 概述

Invoker Server 使用 Centrifugo 作为实时通信层，通过 RPC 调用处理游戏操作。本文档提供 API 索引和快速入门指南。

## 文档结构

### 通用文档

- **[Centrifugo RPC 通用接口](./common-websocket-api-zh.md)**
  - 连接和认证
  - RPC 调用格式
  - 通用接口（getBalance、getConfig）
  - 频道订阅
  - 错误码
  - 金额格式说明

### 游戏专属 API 文档

#### 即时游戏（Instant Games）

| 游戏 | RPC 方法 | 说明 | 文档 |
|------|----------|------|------|
| Dice | `dice.placeBet` | 经典骰子游戏，预测大于或小于目标值 | [查看](./dice-websocket-api-zh.md) |
| Keno | `keno.placeBet` | 数字彩票游戏，支持 4 种难度模式 | [查看](./keno-websocket-api-zh.md) |
| Plinko | `plinko.placeBet` | 弹珠台游戏，支持 8-16 行配置 | [查看](./plinko-websocket-api-zh.md) |
| Limbo | `limbo.placeBet` | 倍率预测游戏，目标倍率 1.01-1,000,000 | [查看](./limbo-websocket-api-zh.md) |
| Dragon Tiger | `dragonTiger.placeBet` | 龙虎斗卡牌游戏，三种投注类型 | [查看](./dragontiger-websocket-api-zh.md) |

#### 多人实时游戏（Multiplayer Real-time Games）

| 游戏 | RPC 方法 | 说明 | 文档 |
|------|----------|------|------|
| Crash | `crash.placeBet`、`crash.cashOut`、`crash.getState` | 多人实时倍率游戏 | [查看](./crash-websocket-api-zh.md) |

#### 会话型游戏（Session Games）

| 游戏 | RPC 方法 | 说明 | 文档 |
|------|----------|------|------|
| Mines | `mines.placeBet`、`mines.reveal`、`mines.cashOut` | 扫雷游戏 | [查看](./mines-websocket-api-zh.md) |
| HiLo | `hilo.placeBet`、`hilo.makeChoice`、`hilo.cashOut` | 高低牌游戏 | [查看](./hilo-websocket-api-zh.md) |
| Chicken Road | `chickenroad.placeBet`、`chickenroad.move`、`chickenroad.cashOut` | 闯关游戏 | [查看](./chickenroad-websocket-api-zh.md) |
| Blackjack | `blackjack.placeBet`、`blackjack.hit`、`blackjack.stand` | 21 点游戏 | [查看](./blackjack-websocket-api-zh.md) |

## 快速开始

### 1. 安装 Centrifuge 客户端

```html
<script src="https://cdn.jsdelivr.net/npm/centrifuge@5.5.2/dist/centrifuge.min.js"></script>
```

### 2. 创建会话获取 Token

通过 Provider API 创建游戏会话：

```javascript
const response = await fetch('https://dev.hicasino.xyz/sessions', {
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
```

### 3. 连接 Centrifugo

```javascript
const centrifuge = new Centrifuge(
    'wss://centrifugo.dev.hicasino.xyz/connection/websocket',
    { data: { token: token } }
);

centrifuge.on('connected', (ctx) => {
    console.log('已连接:', ctx.client);
});

centrifuge.connect();
```

### 4. 调用游戏 RPC

```javascript
// 获取余额
const balance = await centrifuge.rpc('common.getBalance', {});
console.log('余额:', balance.data.balance);

// Dice 下注
const result = await centrifuge.rpc('dice.placeBet', {
    clientSeed: 'my_seed_123',
    amount: '10.00',
    params: { target: 50, isRollOver: true }
});
console.log('结果:', result.data);
```

## RPC 方法命名规范

```
{namespace}.{action}
```

| 命名空间 | 说明 | 示例 |
|----------|------|------|
| `common` | 通用操作 | `common.getBalance`、`common.getConfig` |
| `dice` | Dice 游戏 | `dice.placeBet` |
| `mines` | Mines 游戏 | `mines.placeBet`、`mines.reveal`、`mines.cashOut` |
| `crash` | Crash 游戏 | `crash.placeBet`、`crash.cashOut`、`crash.getState` |

## RESTful API 文档

所有 RESTful API 接口文档已通过 Redoc 自动生成：

**文档地址**: https://storage.googleapis.com/speedix-invoker-api-docs/index.html

### 公平性验证 API

| 游戏 | 端点 | 说明 |
|------|------|------|
| Dice | `POST /v1/fairness/dice/verify` | 验证骰子结果 |
| Mines | `POST /v1/fairness/mines/verify` | 验证地雷位置 |
| Keno | `POST /v1/fairness/keno/verify` | 验证抽奖结果 |
| Plinko | `POST /v1/fairness/plinko/verify` | 验证落球路径 |
| ChickenRoad | `POST /v1/fairness/chickenroad/verify` | 验证每步结果 |

## 实现状态

| 游戏 | 类型 | 状态 | 文档 |
|------|------|------|------|
| Dice | 即时游戏 | ✅ 已实现 | [查看文档](./dice-websocket-api-zh.md) |
| Keno | 即时游戏 | ✅ 已实现 | [查看文档](./keno-websocket-api-zh.md) |
| Plinko | 即时游戏 | ✅ 已实现 | [查看文档](./plinko-websocket-api-zh.md) |
| Limbo | 即时游戏 | ✅ 已实现 | [查看文档](./limbo-websocket-api-zh.md) |
| Dragon Tiger | 即时游戏 | ✅ 已实现 | [查看文档](./dragontiger-websocket-api-zh.md) |
| Crash | 多人实时游戏 | ✅ 已实现 | [查看文档](./crash-websocket-api-zh.md) |
| Mines | 会话游戏 | ✅ 已实现 | [查看文档](./mines-websocket-api-zh.md) |
| HiLo | 会话游戏 | ✅ 已实现 | [查看文档](./hilo-websocket-api-zh.md) |
| Chicken Road | 会话游戏 | ✅ 已实现 | [查看文档](./chickenroad-websocket-api-zh.md) |
| Blackjack | 会话游戏 | ✅ 已实现 | [查看文档](./blackjack-websocket-api-zh.md) |

## 相关文档

- [系统架构](./architecture-zh.md)
- [序列图](./sequence-diagrams-zh.md)

## 测试页面

项目提供测试页面用于调试：

| 页面 | 说明 |
|------|------|
| `public/test_websocket_instant.html` | 即时游戏测试 |
| `public/test_websocket_session.html` | 会话游戏测试 |
| `public/test_websocket_crash.html` | Crash 游戏测试 |

使用方式：
```
http://localhost:8000/test_websocket_instant.html?env=local
http://localhost:8000/test_websocket_instant.html?env=dev
```
