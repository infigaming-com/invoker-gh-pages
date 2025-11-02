# Chicken Road 游戏 WebSocket API 文档

## 游戏概述

Chicken Road 是一款 Crash 类游戏，玩家控制小鸡过马路，每走一步倍率增加。游戏的挑战在于决定何时停止前进并提现——走得越远，倍率越高，但失败的风险也越大。

## 游戏特性

- **会话型游戏**：支持逐步前进，可随时提现
- **四种难度**：easy、medium、hard、daredevil
- **动态倍率**：倍率通过公式动态计算，保证 98% RTP
- **动态概率**：每一步的成功概率基于难度和进度
- **断线重连**：支持游戏恢复功能
- **可证明公平**：完整的 Provably Fair 机制

## 1. PLACE_BET - 开始游戏

使用统一的 PLACE_BET 接口开始新的 Chicken Road 游戏。每个玩家同时只能有一个活跃的游戏。

### 请求消息

```json
{
  "i": "msg_801",
  "t": "PLACE_BET",
  "p": {
    "@type": "type.googleapis.com/game.v1.PlaceBetRequest",
    "gameId": "inhousegame:chickenroad",
    "amount": "10.00",
    "gameParams": {
      "chickenRoad": {
        "difficulty": "easy"
      }
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |
| `difficulty` | string | 是 | 难度等级：easy、medium、hard、daredevil |

### 难度配置

| 难度 | 步数 | RTP | 特点 |
|------|------|-----|------|
| Easy | 24 | 98.5% | 步数多，适合稳健玩家 |
| Medium | 22 | 97.5% | 中等步数，平衡风险与回报 |
| Hard | 20 | 96.5% | 较少步数，高风险高回报 |
| Daredevil | 15 | 96.0% | 极限挑战，倍率增长极快 |

**倍率计算说明**：
- 倍率通过公式动态计算：`倍率 = (1 ÷ 获胜率) × 0.98`
- 获胜率公式：`获胜率 = (总步数 - 当前步数 + 1) / (20 - 当前步数 + 1) × 上一个获胜率`
- 所有难度统一 RTP 为 98%

### 响应消息

```json
{
  "i": "msg_801",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentStep": 0,
    "maxSteps": 19,
    "difficulty": "easy",
    "currentMultiplier": "1.0000",
    "nextMultiplier": "1.0300",
    "nextProbability": 0.95,
    "completedSteps": []
  }
}
```

注：如果已有活跃游戏，将返回错误码 `ACTIVE_SESSION_EXISTS`

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

## 2. CHICKENROAD_MOVE - 前进一步

控制小鸡前进一步，系统判定是否生存。

### 请求消息

```json
{
  "i": "msg_802",
  "t": "CHICKENROAD_MOVE",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息（生存）

```json
{
  "i": "msg_802",
  "t": "CHICKENROAD_MOVE_RESPONSE",
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
  "t": "CHICKENROAD_MOVE_RESPONSE",
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

## 3. CHICKENROAD_CASHOUT - 提现

在至少完成一步后提现当前赢利。

### 请求消息

```json
{
  "i": "msg_803",
  "t": "CHICKENROAD_CASHOUT",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_803",
  "t": "CHICKENROAD_CASHOUT_RESPONSE",
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
    "roundId": "123456789012345678"
  }
}
```

### 响应消息

```json
{
  "i": "msg_806",
  "t": "CHICKENROAD_RESUME_RESPONSE",
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

## 10. 游戏配置获取

### GET_GAME_CONFIG 请求

获取 Chicken Road 游戏的完整配置信息，包括所有难度的倍率表。

```json
{
  "i": "msg_config_001",
  "t": "GET_GAME_CONFIG",
  "p": {
    "gameId": "inhousegame:chickenroad"
  }
}
```

### GET_GAME_CONFIG 响应

```json
{
  "i": "msg_config_001",
  "t": "GET_GAME_CONFIG_RESPONSE",
  "p": {
    "configs": [
      {
        "gameId": "inhousegame:chickenroad",
        "config": {
          "gameId": "inhousegame:chickenroad",
          "gameName": "Chicken Road",
          "category": "session",
          "description": "Guide the chicken across the road",
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 1,
              "minBet": 0.1,
              "maxBet": 100000,
              "maxProfit": 100000
            }
          ],
          "gameParameters": {
            "difficulties": {
              "easy": {
                "maxSteps": 24,
                "rtp": 98,
                "maxMultiplier": 19.44,
                "multipliers": [
                  {"step": 1, "multiplier": 1.03},
                  {"step": 2, "multiplier": 1.08},
                  {"step": 3, "multiplier": 1.15},
                  {"step": 4, "multiplier": 1.25},
                  {"step": 5, "multiplier": 1.37},
                  ...
                  {"step": 24, "multiplier": 19.44}
                ]
              },
              "medium": {
                "maxSteps": 22,
                "rtp": 98,
                "maxMultiplier": 1788.8,
                "multipliers": [...]
              },
              "hard": {
                "maxSteps": 20,
                "rtp": 98,
                "maxMultiplier": 41321.43,
                "multipliers": [...]
              },
              "daredevil": {
                "maxSteps": 15,
                "rtp": 98,
                "maxMultiplier": 2542251.93,
                "multipliers": [...]
              }
            },
            "availableDifficulties": ["easy", "medium", "hard", "daredevil"],
            "defaultDifficulty": "easy"
          },
          "defaultRtp": "98.0%"
        }
      }
    ]
  }
}
```

### 配置响应说明

#### multipliers 字段
`multipliers` 数组包含该难度下所有步骤的倍率信息：
- `step`: 步数（1 到 maxSteps）
- `multiplier`: 达到该步时的倍率
- 前端可以使用此数据直接显示每一步的倍率，无需自己实现复杂的计算公式

#### 使用场景
1. **游戏界面显示**：在游戏开始前显示完整的倍率序列（如截图中的 1.15x, 1.37x, 1.64x...）
2. **策略辅助**：帮助玩家制定前进策略
3. **数据一致性**：确保前后端倍率计算完全一致

## 11. 公平性验证 API

除了 WebSocket 接口外，系统还提供 RESTful API 用于独立验证游戏结果的公平性。

### 11.1 验证接口

**端点**: `POST /v1/fairness/chickenroad/verify`

**认证**: 需要 JWT Token

**请求参数**:

```json
{
  "difficulty": "medium",
  "clientSeed": "player_seed_123",
  "serverSeed": "revealed_server_seed",
  "nonce": 1
}
```

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `difficulty` | string | 是 | 难度等级："easy"、"medium"、"hard"、"expert" |
| `clientSeed` | string | 是 | 客户端种子（游戏时提供的） |
| `serverSeed` | string | 是 | 服务器种子（游戏结束后揭示的） |
| `nonce` | number | 是 | Nonce 值（≥0） |

**响应结果**:

```json
{
  "stepResults": [true, true, false, true, true, false],
  "multipliers": [1.15, 1.37, 1.64, 1.95, 2.32, 2.77]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `stepResults` | boolean[] | 每步的结果（true=成功，false=失败） |
| `multipliers` | number[] | 每步对应的倍率 |

### 11.2 验证步骤

1. **获取游戏数据**：
   - 从游戏结果中获取 `provablyFair` 信息
   - 记录自己提供的 `clientSeed`
   - 记录游戏的 `difficulty`

2. **调用验证接口**：
   ```bash
   curl -X POST https://dev.hicasino.xyz/v1/fairness/chickenroad/verify \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_JWT_TOKEN" \
     -d '{
       "difficulty": "medium",
       "clientSeed": "your_client_seed",
       "serverSeed": "revealed_server_seed",
       "nonce": 1
     }'
   ```

3. **对比结果**：
   - 验证返回的 `stepResults` 与游戏实际步骤结果是否一致
   - 验证返回的 `multipliers` 与游戏中的倍率是否一致
   - 如果全部一致，证明游戏结果公平

### 11.3 JavaScript 验证示例

```javascript
async function verifyChickenRoadResult(gameResult, jwtToken) {
  const response = await fetch('https://dev.hicasino.xyz/v1/fairness/chickenroad/verify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${jwtToken}`
    },
    body: JSON.stringify({
      difficulty: gameResult.difficulty,
      clientSeed: gameResult.provablyFair.clientSeed,
      serverSeed: gameResult.provablyFair.serverSeed,
      nonce: gameResult.provablyFair.nonce
    })
  });

  const verification = await response.json();

  // 对比步骤结果
  const stepsMatch = verification.stepResults.every((result, index) =>
    result === gameResult.stepResults[index]
  );

  // 对比倍率（允许小的浮点误差）
  const multipliersMatch = verification.multipliers.every((mult, index) =>
    Math.abs(mult - gameResult.multipliers[index]) < 0.01
  );

  const isValid = stepsMatch && multipliersMatch;
  console.log('验证结果:', isValid ? '✅ 公平' : '❌ 不匹配');
  return isValid;
}
```

### 11.4 验证原理

验证接口使用与游戏相同的算法生成每步的结果：

```
对于每一步 i（i = 0 到 maxSteps-1）：
  1. 组合种子：seedStr = "serverSeed:clientSeed:nonce:i"
  2. 计算哈希：hash = SHA256(seedStr)
  3. 转换为随机数：将哈希的前 8 位十六进制转换为浮点数（0-1）
  4. 判断结果：
     - randomValue = hashDecimal / (2^32 - 1)
     - survivalRate = 难度配置中的存活率
     - stepResult = randomValue < survivalRate ? true : false
  5. 计算倍率：根据步数和难度从配置表中获取对应倍率
```

**难度配置存活率**：
- Easy: 67% (maxSteps=24)
- Medium: 63% (maxSteps=22)
- Hard: 60% (maxSteps=20)
- Expert: 54% (maxSteps=15)

这确保了：
- **确定性**：相同的种子组合总是产生相同的结果
- **不可预测性**：在服务器种子揭示前无法预测结果
- **可验证性**：任何人都可以独立验证结果

## 12. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Chicken Road 游戏详细设计](./chickenroad-detailed-design-zh.md)