# HiLo 游戏 WebSocket API 文档

## 游戏概述

HiLo 是一款经典的高低牌游戏。玩家需要预测下一张牌是比当前牌高（Higher）还是低（Lower）。每次猜对，赔率累积增加；猜错则失去全部赌注。玩家可以在任何时候选择提现当前赢利。

## 游戏特性

- **会话型游戏**：支持多轮猜测，赔率累积
- **动态赔率**：根据概率实时计算每个选择的赔率
- **灵活提现**：至少猜对一次后可随时提现
- **跳过功能**：开始时可跳过不喜欢的起始牌
- **断线重连**：支持游戏恢复功能
- **可证明公平**：完整的 Provably Fair 机制

## 1. HILO_START_GAME - 开始游戏

开始新的 HiLo 游戏。每个玩家同时只能有一个活跃的 HiLo 游戏。

### 请求消息

```json
{
  "i": "msg_601",
  "t": "HILO_START_GAME",
  "p": {
    "amount": "10.00"  // 可选：空字符串""或"0"表示试玩模式
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |

### 响应消息

```json
{
  "i": "msg_601",
  "t": "HILO_START_GAME_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentCard": 7,
    "cardHistory": [7],
    "currentMultiplier": "1.0000",
    "multiplierHistory": [],
    "canCashout": false,
    "guessCount": 0,
    "higherProbability": 0.4615,
    "lowerProbability": 0.4615,
    "higherMultiplier": "2.1450",
    "lowerMultiplier": "2.1450"
  }
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID，纯数字格式 |
| `status` | string | 游戏状态：playing、finished、cashed_out |
| `betAmount` | string | 下注金额（decimal 类型，字符串格式） |
| `currentCard` | number | 当前牌面值（1-13） |
| `cardHistory` | number[] | 历史牌面记录 |
| `currentMultiplier` | string | 当前累积赔率 |
| `multiplierHistory` | number[] | 历史赔率记录 |
| `canCashout` | boolean | 是否可以提现（至少猜对一次后为 true） |
| `guessCount` | number | 已猜测次数 |
| `higherProbability` | number | 下张牌更高的概率 |
| `lowerProbability` | number | 下张牌更低的概率 |
| `higherMultiplier` | string | 选择"higher"的赔率 |
| `lowerMultiplier` | string | 选择"lower"的赔率 |

## 2. HILO_MAKE_CHOICE - 做出选择

预测下一张牌的高低。

### 请求消息

```json
{
  "i": "msg_602",
  "t": "HILO_MAKE_CHOICE",
  "p": {
    "roundId": "123456789012345678",
    "choice": "higher"  // 可选值: "higher", "lower", "skip", "same"
  }
}
```

### 选择说明

| 选择 | 说明 | 使用条件 |
|------|------|----------|
| `higher` | 预测下一张牌更大 | 当前牌不是K |
| `lower` | 预测下一张牌更小 | 当前牌不是A |
| `skip` | 跳过当前牌，换一张起始牌 | 仅在未做任何猜测前可用 |
| `same` | 预测相同 | 自动应用于边界情况 |

### 响应消息（猜对）

```json
{
  "i": "msg_602",
  "t": "HILO_MAKE_CHOICE_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "choice": "higher",
    "guessCorrect": true,
    "status": "playing",
    "currentCard": 10,
    "cardHistory": [7, 10],
    "currentMultiplier": "2.1450",
    "multiplierHistory": [2.1450],
    "canCashout": true,
    "guessCount": 1,
    "higherProbability": 0.2308,
    "lowerProbability": 0.6923,
    "higherMultiplier": "4.2893",
    "lowerMultiplier": "1.4298"
  }
}
```

### 响应消息（猜错）

```json
{
  "i": "msg_602",
  "t": "HILO_MAKE_CHOICE_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "choice": "higher",
    "guessCorrect": false,
    "status": "finished",
    "currentCard": 3,
    "cardHistory": [7, 3],
    "currentMultiplier": "0.0000",
    "finalPayout": "0.00000000",
    "isProfitable": false
  }
}
```

## 3. HILO_CASH_OUT - 兑现

在至少猜对一次后兑现当前赢利。

### 请求消息

```json
{
  "i": "msg_603",
  "t": "HILO_CASH_OUT",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_603",
  "t": "HILO_CASH_OUT_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "cashed_out",
    "payout": "21.450000",
    "cashedOut": true,
    "finalPayout": "21.45000000",
    "isProfitable": true,
    "currentMultiplier": "2.1450"
  }
}
```

## 4. HILO_GET_STATE - 获取游戏状态

获取指定回合的游戏状态。

### 请求消息

```json
{
  "i": "msg_604",
  "t": "HILO_GET_STATE",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_604",
  "t": "HILO_GET_STATE_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentCard": 10,
    "cardHistory": [7, 10],
    "currentMultiplier": "2.1450",
    "multiplierHistory": [2.1450],
    "canCashout": true,
    "guessCount": 1
  }
}
```

## 5. HILO_CHECK_ACTIVE - 检查活跃游戏

检查玩家是否有未完成的游戏。

### 请求消息

```json
{
  "i": "msg_605",
  "t": "HILO_CHECK_ACTIVE",
  "p": {}
}
```

### 响应消息（有活跃游戏）

```json
{
  "i": "msg_605",
  "t": "HILO_CHECK_ACTIVE_RESPONSE",
  "p": {
    "hasActiveGame": true,
    "roundId": "123456789012345678",
    "gameState": {
      "status": "playing",
      "currentCard": 7,
      "cardHistory": [7],
      "currentMultiplier": "1.0000",
      "canCashout": false
    }
  }
}
```

### 响应消息（无活跃游戏）

```json
{
  "i": "msg_605",
  "t": "HILO_CHECK_ACTIVE_RESPONSE",
  "p": {
    "hasActiveGame": false
  }
}
```

## 6. HILO_RESUME_GAME - 恢复游戏

恢复未完成的游戏。

### 请求消息

```json
{
  "i": "msg_606",
  "t": "HILO_RESUME_GAME",
  "p": {
    "roundId": "123456789012345678"  // 可选，不提供则恢复最新的活跃游戏
  }
}
```

### 响应消息

```json
{
  "i": "msg_606",
  "t": "HILO_RESUME_GAME_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "resumed": true,
    "status": "playing",
    "currentCard": 7,
    "cardHistory": [7],
    "currentMultiplier": "1.0000",
    "canCashout": false
  }
}
```

## 7. 游戏规则

### 牌面值对应

| 牌面 | 数值 | 说明 |
|------|------|------|
| A | 1 | 最小牌 |
| 2-10 | 2-10 | 数字牌 |
| J | 11 | Jack |
| Q | 12 | Queen |
| K | 13 | 最大牌 |

### 赔率计算

赔率基于概率动态计算，确保 99% 的 RTP：

```
赔率 = 0.99 / 获胜概率
```

### 特殊规则

1. **边界处理**
   - 当前牌为 A 时，选择 "lower" 自动变为 "same"
   - 当前牌为 K 时，选择 "higher" 自动变为 "same"

2. **提现条件**
   - 必须至少猜对一次才能提现
   - 累积赔率 = 所有猜对的赔率相乘

3. **跳过功能**
   - 仅在游戏开始且未做任何猜测时可用
   - 用于更换不喜欢的起始牌

### 概率示例

| 当前牌 | Higher概率 | Lower概率 | Higher赔率 | Lower赔率 |
|--------|------------|-----------|------------|-----------|
| A(1) | 92.3% | 0% | 1.07x | - |
| 7 | 46.2% | 46.2% | 2.14x | 2.14x |
| Q(12) | 7.7% | 84.6% | 12.86x | 1.17x |
| K(13) | 0% | 92.3% | - | 1.07x |

## 8. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `ACTIVE_SESSION_EXISTS` | 已有进行中的游戏 | 结束或恢复当前游戏 |
| `GAME_NOT_FOUND` | 游戏不存在 | 检查 roundId 是否正确 |
| `INVALID_CHOICE` | 无效的选择 | 使用有效的选择值 |
| `CASHOUT_FAILED` | 兑现失败 | 确保至少猜对一次 |
| `NO_ACTIVE_GAME` | 没有活跃的游戏 | 先开始新游戏 |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 通过 Seed Service API 设置客户端种子 |

## 9. 注意事项

1. **单游戏限制**：每个玩家同时只能有一个活跃的 HiLo 游戏
2. **提现条件**：必须至少猜对一次才能提现
3. **边界处理**：A 和 K 的特殊处理规则
4. **跳过功能**：仅在游戏开始时可用
5. **断线恢复**：支持通过 HILO_RESUME_GAME 恢复游戏
6. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成
7. **累积赔率**：每次猜对的赔率会累积相乘

## 10. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [HiLo 游戏详细设计](./hilo-detailed-design-zh.md)