# Dice 游戏 WebSocket API 文档

## 游戏概述

Dice 是一款经典的骰子游戏，玩家预测骰子的点数是否会大于或小于设定的目标值。游戏支持 4-96 的目标数字范围，RTP 为 99%。

## 游戏特性

- **即时游戏**：单次下注立即开奖
- **可证明公平**：支持完整的 Provably Fair 机制
- **灵活赔率**：根据目标数字和条件动态计算赔率
- **试玩模式**：支持免费试玩体验
- **目标范围**：4-96（避免极端值导致赔率小于1）

## 1. PLACE_BET - 下注

发起 Dice 游戏下注。支持试玩模式（amount为空或"0"）。

### 请求消息

```json
{
  "i": "msg_007",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",  // 可选：空字符串""或"0"表示试玩模式
    "gameParams": {
      "diceParams": {
        "target": 50,      // 目标数字 (4-96)
        "condition": "OVER" // "OVER" 或 "UNDER"
      }
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |
| `gameParams.diceParams.target` | number | 是 | 目标数字，范围 4-96 |
| `gameParams.diceParams.condition` | string | 是 | 预测条件："OVER"（大于）或 "UNDER"（小于） |

### 响应消息

```json
{
  "i": "msg_007",
  "t": "PLACE_BET",
  "p": {
    "roundId": "123456789012345678",  // 纯数字的回合ID
    "gameResult": {
      "gameId": "inhousegame:dice",
      "betAmount": "10.00",
      "winAmount": "20.00",
      "isWin": true,
      "diceOutcome": {
        "rolledNumber": 75.23,  // 骰子结果（0.00-99.99）
        "target": 50,
        "condition": "OVER",
        "multiplier": "2.00"
      },
      "multiplier": "2.00",
      "timestamp": 1704067200000
    },
    "balance": "1010.50"
  }
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID，纯数字格式 |
| `gameResult.betAmount` | string | 下注金额 |
| `gameResult.winAmount` | string | 获胜金额 |
| `gameResult.isWin` | boolean | 是否获胜 |
| `diceOutcome.rolledNumber` | number | 骰子结果，范围 0.00-99.99 |
| `diceOutcome.target` | number | 目标数字 |
| `diceOutcome.condition` | string | 预测条件 |
| `diceOutcome.multiplier` | string | 游戏赔率 |
| `balance` | string | 更新后的余额 |

## 2. GET_BET_HISTORY - 获取投注历史

获取玩家的 Dice 游戏投注历史记录。

### 请求消息

```json
{
  "i": "msg_008",
  "t": "GET_BET_HISTORY",
  "p": {
    "gameId": "inhousegame:dice",  // 可选，指定游戏类型
    "limit": 10,                    // 可选，默认10
    "offset": 0                     // 可选，默认0
  }
}
```

### 响应消息

```json
{
  "i": "msg_008",
  "t": "GET_BET_HISTORY",
  "p": {
    "history": [
      {
        "roundId": "123456789012345678",
        "gameId": "inhousegame:dice",
        "betAmount": "10.00",
        "winAmount": "20.00",
        "isWin": true,
        "multiplier": "2.00",
        "diceOutcome": {
          "rolledNumber": 75.23,
          "target": 50,
          "condition": "OVER"
        },
        "timestamp": 1704067200000
      }
    ],
    "total": 100,
    "hasMore": true
  }
}
```

## 3. 游戏规则

### 赔率计算

赔率计算基于概率，确保 99% 的 RTP：

- **OVER 条件**：`赔率 = 99 / (99.99 - target)`
- **UNDER 条件**：`赔率 = 99 / target`

### 示例

1. **目标 50，预测 OVER**
   - 获胜概率：49.99%
   - 赔率：1.98x
   
2. **目标 10，预测 UNDER**
   - 获胜概率：10%
   - 赔率：9.9x

3. **目标 95，预测 OVER**
   - 获胜概率：4.99%
   - 赔率：19.84x

## 4. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `INVALID_TARGET` | 目标数字超出范围 | 使用 4-96 范围内的数字 |
| `INVALID_CONDITION` | 无效的预测条件 | 使用 "OVER" 或 "UNDER" |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 通过 Seed Service API 设置客户端种子 |
| `INSUFFICIENT_BALANCE` | 余额不足 | 充值或减少下注金额 |
| `BET_LIMIT_EXCEEDED` | 超出投注限额 | 调整下注金额 |

## 5. 注意事项

1. **目标数字范围**：必须在 4-96 之间，避免极端值导致赔率问题
2. **客户端种子**：通过 Seed Service API 管理，游戏自动使用已设置的种子
3. **试玩模式**：将 amount 设为 "0" 或空字符串即可免费试玩
4. **精度处理**：所有金额使用字符串格式，保留8位小数
5. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成

## 6. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Dice 游戏详细设计](./dice-detailed-design-zh.md)