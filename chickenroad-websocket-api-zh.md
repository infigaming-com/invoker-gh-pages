# Chicken Road 游戏 WebSocket API 文档

## 游戏概述

Chicken Road 是一款 Crash 类游戏，玩家控制小鸡过马路，每走一步倍率增加。游戏的挑战在于决定何时停止前进并提现——走得越远，倍率越高，但失败的风险也越大。

## 游戏特性

- **会话型游戏**：支持逐步前进，可随时提现
- **四种难度**：easy、medium、hard、daredevil
- **超高倍率**：最高可达 2,542,251.93 倍（daredevil难度）
- **动态概率**：每一步的成功概率基于难度和进度
- **断线重连**：支持游戏恢复功能
- **可证明公平**：完整的 Provably Fair 机制

## 1. CHICKENROAD_START_GAME - 开始游戏

开始新的 Chicken Road 游戏。每个玩家同时只能有一个活跃的游戏。

### 请求消息

```json
{
  "i": "msg_801",
  "t": "CHICKENROAD_START_GAME",
  "p": {
    "amount": "10.00",  // 可选：空字符串""或"0"表示试玩模式
    "difficulty": "easy"  // 难度等级
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |
| `difficulty` | string | 是 | 难度等级：easy、medium、hard、daredevil |

### 难度配置

| 难度 | 步数 | 最高倍率 | RTP | 特点 |
|------|------|----------|-----|------|
| Easy | 24 | 19.44x | 98.5% | 稳定小赢，适合新手 |
| Medium | 22 | 1,788.80x | 97.5% | 平衡风险与回报 |
| Hard | 20 | 41,321.43x | 96.5% | 高风险高回报 |
| Daredevil | 15 | 2,542,251.93x | 96.0% | 极限挑战，巨额奖金 |

### 响应消息

```json
{
  "i": "msg_801",
  "t": "CHICKENROAD_START_GAME_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentStep": 0,
    "maxSteps": 24,
    "difficulty": "easy",
    "currentMultiplier": "1.0000",
    "nextMultiplier": "1.0300",
    "nextProbability": 0.985,
    "completedSteps": []
  }
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID，纯数字格式 |
| `status` | string | 游戏状态：playing、finished、cashed_out |
| `currentStep` | number | 当前所在步数（0表示起点） |
| `maxSteps` | number | 最大步数（根据难度） |
| `currentMultiplier` | string | 当前累积倍率 |
| `nextMultiplier` | string | 下一步的倍率 |
| `nextProbability` | number | 下一步成功的概率 |
| `completedSteps` | number[] | 已完成的步数列表 |

## 2. CHICKENROAD_MOVE_FORWARD - 前进一步

控制小鸡前进一步，系统判定是否生存。

### 请求消息

```json
{
  "i": "msg_802",
  "t": "CHICKENROAD_MOVE_FORWARD",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息（生存）

```json
{
  "i": "msg_802",
  "t": "CHICKENROAD_MOVE_FORWARD_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "survived": true,
    "status": "playing",
    "currentStep": 1,
    "maxSteps": 24,
    "currentMultiplier": "1.0300",
    "nextMultiplier": "1.0700",
    "nextProbability": 0.963,
    "completedSteps": [0],
    "finalPayout": "0.00000000",
    "isProfitable": false
  }
}
```

### 响应消息（失败）

```json
{
  "i": "msg_802",
  "t": "CHICKENROAD_MOVE_FORWARD_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "survived": false,
    "status": "finished",
    "currentStep": 1,
    "maxSteps": 24,
    "currentMultiplier": "0.0000",
    "completedSteps": [0],
    "finalPayout": "0.00000000",
    "isProfitable": false
  }
}
```

## 3. CHICKENROAD_CASH_OUT - 提现

在至少完成一步后提现当前赢利。

### 请求消息

```json
{
  "i": "msg_803",
  "t": "CHICKENROAD_CASH_OUT",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_803",
  "t": "CHICKENROAD_CASH_OUT_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "cashed_out",
    "payout": "15.30000000",
    "cashedOut": true,
    "finalPayout": "15.30000000",
    "isProfitable": true,
    "currentMultiplier": "1.5300",
    "completedSteps": [0, 1, 2, 3, 4, 5, 6, 7, 8]
  }
}
```

## 4. CHICKENROAD_GET_STATE - 获取游戏状态

获取指定回合的游戏状态。

### 请求消息

```json
{
  "i": "msg_804",
  "t": "CHICKENROAD_GET_STATE",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_804",
  "t": "CHICKENROAD_GET_STATE_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentStep": 5,
    "maxSteps": 24,
    "difficulty": "easy",
    "currentMultiplier": "1.2900",
    "nextMultiplier": "1.3600",
    "nextProbability": 0.949,
    "completedSteps": [0, 1, 2, 3, 4],
    "survived": true,
    "cashedOut": false
  }
}
```

## 5. CHICKENROAD_CHECK_ACTIVE - 检查活跃游戏

检查玩家是否有未完成的游戏。

### 请求消息

```json
{
  "i": "msg_805",
  "t": "CHICKENROAD_CHECK_ACTIVE",
  "p": {}
}
```

### 响应消息（有活跃游戏）

```json
{
  "i": "msg_805",
  "t": "CHICKENROAD_CHECK_ACTIVE_RESPONSE",
  "p": {
    "hasActiveGame": true,
    "roundId": "123456789012345678",
    "gameState": {
      "status": "playing",
      "currentStep": 5,
      "maxSteps": 24,
      "difficulty": "easy",
      "currentMultiplier": "1.2900",
      "completedSteps": [0, 1, 2, 3, 4],
      "survived": true
    }
  }
}
```

### 响应消息（无活跃游戏）

```json
{
  "i": "msg_805",
  "t": "CHICKENROAD_CHECK_ACTIVE_RESPONSE",
  "p": {
    "hasActiveGame": false
  }
}
```

## 6. CHICKENROAD_RESUME_GAME - 恢复游戏

恢复未完成的游戏。

### 请求消息

```json
{
  "i": "msg_806",
  "t": "CHICKENROAD_RESUME_GAME",
  "p": {
    "roundId": "123456789012345678"  // 可选，不提供则恢复最新的活跃游戏
  }
}
```

### 响应消息

```json
{
  "i": "msg_806",
  "t": "CHICKENROAD_RESUME_GAME_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "resumed": true,
    "status": "playing",
    "currentStep": 5,
    "maxSteps": 24,
    "difficulty": "easy",
    "currentMultiplier": "1.2900",
    "nextMultiplier": "1.3600",
    "nextProbability": 0.949,
    "completedSteps": [0, 1, 2, 3, 4],
    "survived": true,
    "cashedOut": false
  }
}
```

## 7. 游戏规则

### 倍率序列示例

#### Easy 难度（24步）
| 步数 | 倍率 | 概率 |
|------|------|------|
| 1 | 1.03x | 98.5% |
| 5 | 1.29x | 94.9% |
| 10 | 2.13x | 89.6% |
| 15 | 4.82x | 79.0% |
| 20 | 11.99x | 63.3% |
| 24 | 19.44x | 50.5% |

#### Daredevil 难度（15步）
| 步数 | 倍率 | 概率 |
|------|------|------|
| 1 | 1.60x | 60.0% |
| 3 | 6.40x | 37.5% |
| 5 | 39.00x | 24.6% |
| 7 | 470.23x | 16.2% |
| 10 | 17,074.29x | 9.1% |
| 15 | 2,542,251.93x | 3.8% |

### 游戏机制

1. **概率计算**
   - 每步的成功概率基于难度和当前进度
   - 概率逐步降低，风险递增

2. **提现条件**
   - 必须至少完成一步才能提现
   - 达到最大步数时自动提现

3. **失败判定**
   - 使用 Provably Fair 算法生成随机数
   - 根据概率判定是否生存

## 8. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `ACTIVE_SESSION_EXISTS` | 已有进行中的游戏 | 结束或恢复当前游戏 |
| `INVALID_AMOUNT` | 无效的下注金额 | 检查金额格式 |
| `INVALID_DIFFICULTY` | 无效的难度参数 | 使用有效的难度值 |
| `MOVE_FAILED` | 前进失败 | 检查游戏状态 |
| `CASHOUT_FAILED` | 提现失败 | 确保至少完成一步 |
| `GAME_NOT_FOUND` | 游戏不存在 | 检查 roundId |
| `NO_ACTIVE_GAME` | 没有活跃的游戏 | 先开始新游戏 |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 通过 Seed Service API 设置客户端种子 |


## 9. 注意事项

1. **单游戏限制**：每个玩家同时只能有一个活跃的 Chicken Road 游戏
2. **提现条件**：必须至少完成一步才能提现
3. **自动提现**：达到最大步数时自动提现
4. **难度选择**：根据风险承受能力选择合适的难度
5. **概率递减**：随着步数增加，成功概率逐步降低
6. **断线恢复**：支持通过 CHICKENROAD_RESUME_GAME 恢复游戏
7. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成

## 10. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Chicken Road 游戏详细设计](./chickenroad-detailed-design-zh.md)