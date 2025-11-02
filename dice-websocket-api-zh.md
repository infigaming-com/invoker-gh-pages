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

## 6. 公平性验证 API

除了 WebSocket 接口外，系统还提供 RESTful API 用于独立验证游戏结果的公平性。

### 6.1 验证接口

**端点**: `POST /v1/fairness/dice/verify`

**认证**: 需要 JWT Token

**请求参数**:

```json
{
  "clientSeed": "player_seed_123",
  "serverSeed": "revealed_server_seed",
  "nonce": 1
}
```

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `clientSeed` | string | 是 | 客户端种子（游戏时提供的） |
| `serverSeed` | string | 是 | 服务器种子（游戏结束后揭示的） |
| `nonce` | number | 是 | Nonce 值（≥0） |

**响应结果**:

```json
{
  "roll": 53.42857142
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `roll` | number | 骰子掷出的值（0-100） |

### 6.2 验证步骤

1. **获取游戏数据**：
   - 从游戏结果中获取 `provablyFair` 信息
   - 记录自己提供的 `clientSeed`

2. **调用验证接口**：
   ```bash
   curl -X POST https://dev.hicasino.xyz/v1/fairness/dice/verify \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_JWT_TOKEN" \
     -d '{
       "clientSeed": "your_client_seed",
       "serverSeed": "revealed_server_seed",
       "nonce": 1
     }'
   ```

3. **对比结果**：
   - 验证返回的 `roll` 值与游戏结果中的掷骰值是否一致
   - 如果一致，证明游戏结果公平

### 6.3 JavaScript 验证示例

```javascript
async function verifyDiceResult(gameResult, jwtToken) {
  const response = await fetch('https://dev.hicasino.xyz/v1/fairness/dice/verify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${jwtToken}`
    },
    body: JSON.stringify({
      clientSeed: gameResult.provablyFair.clientSeed,
      serverSeed: gameResult.provablyFair.serverSeed,
      nonce: gameResult.provablyFair.nonce
    })
  });

  const verification = await response.json();

  // 对比掷骰值（允许小的浮点误差）
  const isValid = Math.abs(verification.roll - gameResult.roll) < 0.00001;

  console.log('验证结果:', isValid ? '✅ 公平' : '❌ 不匹配');
  console.log(`游戏结果: ${gameResult.roll}, 验证结果: ${verification.roll}`);
  return isValid;
}
```

### 6.4 验证原理

验证接口使用与游戏相同的算法生成骰子值：

```
1. 组合种子：seedStr = "serverSeed:clientSeed:nonce"
2. 计算哈希：hash = SHA256(seedStr)
3. 提取前8位十六进制：hashFirst8 = hash[0:8]
4. 转换为十进制：hashDecimal = parseInt(hashFirst8, 16)
5. 归一化到0-100：roll = (hashDecimal / 0xFFFFFFFF) * 100
```

**示例计算**:
```
seedStr = "server123:client456:1"
hash = SHA256(seedStr) = "8a3f2b1c..."
hashFirst8 = "8a3f2b1c"
hashDecimal = 2318840604
roll = (2318840604 / 4294967295) * 100 = 53.98642...
```

这确保了：
- **确定性**：相同的种子组合总是产生相同的结果
- **不可预测性**：在服务器种子揭示前无法预测结果
- **可验证性**：任何人都可以独立验证结果
- **均匀分布**：骰子值在 0-100 范围内均匀分布

## 7. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Dice 游戏详细设计](./dice-detailed-design-zh.md)