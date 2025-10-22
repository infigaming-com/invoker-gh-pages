# WebSocket API 文档索引

## 概述

Invoker Server 的 WebSocket API 文档已按游戏类型拆分为多个独立文档，便于维护和查阅。

## 文档结构

### 通用文档

- **[WebSocket 通用接口](./common-websocket-api-zh.md)**
  - 连接管理（连接、认证、心跳）
  - 通用接口（LOGIN、GET_BALANCE、GET_GAME_CONFIG 等）
  - 事件推送（BALANCE_UPDATE、BET_ACTIVITY 等）
  - 错误处理
  - 金额格式说明

### 游戏专属 API 文档

#### 即时游戏（Instant Games）

- **[Dice 游戏 API](./dice-websocket-api-zh.md)**
  - 经典骰子游戏
  - 预测大于或小于目标值
  - RTP: 99%

- **[Keno 游戏 API](./keno-websocket-api-zh.md)**
  - 数字彩票游戏
  - 从 1-40 选择 1-10 个数字
  - 支持 4 种难度模式
  - RTP: 70-80%

- **[Plinko 游戏 API](./plinko-websocket-api-zh.md)**
  - 弹珠台游戏
  - 支持 8-16 行配置
  - 三种难度等级
  - 最高倍率: 1000x+

- **[Limbo 游戏 API](./limbo-websocket-api-zh.md)**
  - 倍率预测游戏
  - 目标倍率: 1.01-1,000,000.00
  - 指数分布奖励
  - RTP: 99%

- **[Dragon Tiger 游戏 API](./dragontiger-websocket-api-zh.md)**
  - 经典龙虎斗卡牌游戏
  - 三种投注类型（龙/虎/和）
  - 5% 佣金机制
  - RTP: 97.1%

#### 会话型游戏（Session Games）

- **[Mines 游戏 API](./mines-websocket-api-zh.md)**
  - 扫雷游戏
  - 支持 3x3、5x5、7x7 网格
  - 可随时提现
  - 自动提现机制

- **[HiLo 游戏 API](./hilo-websocket-api-zh.md)**
  - 高低牌游戏
  - 预测下一张牌的高低
  - 赔率累积
  - RTP: 99%

- **[Chicken Road 游戏 API](./chickenroad-websocket-api-zh.md)**
  - Crash 类游戏
  - 四种难度等级
  - 最高倍率: 2,542,251.93x
  - RTP: 96-98.5%

- **[Blackjack 游戏 API](./blackjack-websocket-api-zh.md)**
  - 经典 21 点游戏
  - S17 庄家规则
  - 支持分牌、加倍、保险
  - RTP: 99.28%

## RESTful API 文档

所有 RESTful API 接口文档已通过 Redoc 自动生成：
- **文档地址**: https://storage.googleapis.com/speedix-invoker-api-docs/index.html

## 快速开始

### 1. 建立连接

```javascript
const ws = new WebSocket('wss://dev.hicasino.xyz/v1/ws?token=YOUR_JWT_TOKEN');
```

### 2. 选择游戏

根据游戏类型查阅对应的 API 文档：
- 即时游戏：一次下注立即开奖
- 会话型游戏：支持多步操作，可随时提现

### 3. 消息格式

所有消息使用统一格式：
```json
{
  "i": "消息ID",
  "t": "消息类型",
  "p": { /* 消息载荷 */ }
}
```

## 实现状态

| 游戏 | 类型 | 状态 | 文档 |
|------|------|------|------|
| Dice | 即时游戏 | ✅ 已实现 | [查看文档](./dice-websocket-api-zh.md) |
| Mines | 会话游戏 | ✅ 已实现 | [查看文档](./mines-websocket-api-zh.md) |
| Keno | 即时游戏 | ✅ 已实现 | [查看文档](./keno-websocket-api-zh.md) |
| Plinko | 即时游戏 | ✅ 已实现 | [查看文档](./plinko-websocket-api-zh.md) |
| HiLo | 会话游戏 | ✅ 已实现 | [查看文档](./hilo-websocket-api-zh.md) |
| Chicken Road | 会话游戏 | ✅ 已实现 | [查看文档](./chickenroad-websocket-api-zh.md) |
| Blackjack | 会话游戏 | ✅ 已实现 | [查看文档](./blackjack-websocket-api-zh.md) |
| Limbo | 即时游戏 | ✅ 已实现 | [查看文档](./limbo-websocket-api-zh.md) |
| Dragon Tiger | 即时游戏 | ✅ 已实现 | [查看文档](./dragontiger-websocket-api-zh.md) |

## 相关文档

- [系统架构](./architecture-zh.md)
- [详细设计](./detailed-design-zh.md)
- [序列图](./sequence-diagrams-zh.md)

## 联系支持

如有问题或需要帮助，请联系技术支持团队。