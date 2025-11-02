# Plinko 游戏 WebSocket API 文档

## 游戏概述

Plinko 是一款经典的弹珠台游戏。球从顶部落下，经过多排钉子随机弹跳，最终落入底部的槽位中。不同槽位对应不同的赔率，中间槽位概率高但赔率低，边缘槽位概率低但赔率高。

## 游戏特性

- **即时开奖**：单次下注立即开奖
- **延迟结算**：支持动画播放完成后再结算余额
- **可配置行数**：支持 8-16 行
- **三种难度**：low、medium、high 不同风险等级
- **对称赔率**：赔率从中心向两边对称分布
- **高赔率潜力**：最高可达 1000 倍以上（16行高难度）
- **可证明公平**：完整的 Provably Fair 机制

## 1. PLACE_BET - Plinko 下注

发起 Plinko 游戏下注，立即开奖。

### 请求消息

```json
{
  "i": "msg_601",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",  // 可选：空字符串""或"0"表示试玩模式
    "gameParams": {
      "plinko": {
        "rows": 12,               // 行数（8-16）
        "difficulty": "medium"    // 难度（"low", "medium", "high"）
      }
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |
| `plinko.rows` | number | 是 | 游戏板行数，范围 8-16 |
| `plinko.difficulty` | string | 是 | 难度等级：low、medium、high |

### 难度说明

| 难度 | 特点 | 边缘赔率 | 中心赔率 | 适合人群 |
|------|------|----------|----------|----------|
| Low | 低风险，赔率分布平均 | 5-10x | 0.5-1x | 保守玩家 |
| Medium | 中等风险，平衡设计 | 10-30x | 0.3-0.7x | 普通玩家 |
| High | 高风险，极端赔率 | 100-1000x | 0.1-0.3x | 追求刺激玩家 |

### 响应消息

```json
{
  "i": "msg_601",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "gameResult": {
      "gameId": "inhousegame:plinko",
      "betAmount": "10.00",
      "winAmount": "18.00",
      "isWin": true,
      "multiplier": "1.8",
      "timestamp": 1704067200000,
      "plinkoOutcome": {
        "rows": 12,
        "difficulty": "medium",
        "path": [0, 1, 1, 0, 1, 0, 0, 1, 1, 0, 1, 0],
        "finalSlot": 6,
        "multiplier": "1.8"
      }
    },
    "hasAnimation": true
  }
}
```

⚠️ **注意**：Plinko 采用延迟结算机制，响应中**不包含 `balance` 字段**，前端需要在动画播放完成后调用 `SETTLE_BET` 接口获取最终余额。

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID，纯数字格式 |
| `gameResult.betAmount` | string | 下注金额 |
| `gameResult.winAmount` | string | 获胜金额 |
| `gameResult.isWin` | boolean | 是否获胜（赔率>1） |
| `plinkoOutcome.rows` | number | 使用的行数 |
| `plinkoOutcome.difficulty` | string | 使用的难度 |
| `plinkoOutcome.path` | number[] | 球的路径，0=左，1=右 |
| `plinkoOutcome.finalSlot` | number | 最终槽位（0到rows） |
| `plinkoOutcome.multiplier` | string | 槽位对应的赔率 |
| `hasAnimation` | boolean | 是否需要播放动画（Plinko 始终为 true） |

## 2. SETTLE_BET - Plinko 结算

动画播放完成后，调用此接口完成余额结算。

### 请求消息

```json
{
  "i": "msg_602",
  "t": "SETTLE_BET",
  "p": {
    "roundId": "123456789012345678"
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `roundId` | string | 是 | 游戏回合ID（从 PLACE_BET 响应获取） |

### 响应消息

```json
{
  "i": "msg_602",
  "t": "SETTLE_BET_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "balance": "1018.00"
  }
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID |
| `balance` | string | 结算后的最终余额 |

### 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `GAME_NOT_FOUND` | 游戏回合不存在 | 检查 roundId 是否正确 |
| `ALREADY_SETTLED` | 已经结算完成 | 查询当前余额 |
| `INTERNAL_ERROR` | 服务器内部错误 | 重试或联系客服 |

### 注意事项

1. **幂等性**：重复调用 SETTLE_BET 不会重复结算，会返回当前余额
2. **超时自动结算**：如果 20 秒内未调用 SETTLE_BET，服务器会自动结算
3. **前端崩溃恢复**：重连后可以直接调用 SETTLE_BET 完成结算

### 完整流程示例

```javascript
// 1. 下注
const placeBetResponse = await sendMessage({
  t: "PLACE_BET",
  p: {
    amount: "10.00",
    gameParams: { plinko: { rows: 12, difficulty: "medium" } }
  }
});

// 2. 播放动画
await playAnimation(placeBetResponse.gameResult.plinkoOutcome.path);

// 3. 动画完成后结算
const settleBetResponse = await sendMessage({
  t: "SETTLE_BET",
  p: { roundId: placeBetResponse.roundId }
});

// 4. 更新余额显示
updateBalance(settleBetResponse.balance);
```

## 3. 游戏机制

### 路径生成

1. **算法**：使用 SHA-256 哈希算法
2. **输入**：`clientSeed + serverSeed + nonce`
3. **输出**：每一位决定球在该行的方向（0=左，1=右）

### 槽位计算

- 槽位数量 = 行数 + 1
- 最终槽位 = 路径中"1"的数量
- 例如：12行有13个槽位（0-12）

### 概率分布

基于二项分布，中间槽位概率最高：

| 槽位位置 | 概率特点 |
|----------|----------|
| 中心槽位 | 最高概率（约15-20%） |
| 次中心 | 较高概率（约10-15%） |
| 中间区域 | 中等概率（约5-10%） |
| 边缘槽位 | 最低概率（<1%） |

## 4. 赔率表示例

### 12行 Medium 难度

| 槽位 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|------|---|---|---|---|---|---|---|---|---|---|-----|-----|-----|
| 赔率 | 24 | 12 | 4 | 2 | 1.2 | 0.6 | 0.4 | 0.6 | 1.2 | 2 | 4 | 12 | 24 |

### 16行 High 难度

| 槽位 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|------|---|---|---|---|---|---|---|---|---|
| 赔率 | 1000 | 200 | 50 | 10 | 2 | 0.5 | 0.2 | 0.1 | 0.1 |
| 槽位 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | |
| 赔率 | 0.1 | 0.2 | 0.5 | 2 | 10 | 50 | 200 | 1000 | |

## 5. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `INVALID_ROWS` | 行数不在8-16范围内 | 调整行数到有效范围 |
| `INVALID_DIFFICULTY` | 无效的难度参数 | 使用 low、medium 或 high |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 通过 Seed Service API 设置客户端种子 |
| `INSUFFICIENT_BALANCE` | 余额不足 | 充值或减少下注金额 |
| `BET_LIMIT_EXCEEDED` | 超出投注限额 | 调整下注金额 |

## 6. 注意事项

1. **即时游戏**：Plinko 是即时开奖游戏，无需会话管理
2. **行数选择**：行数越多，赔率分布越细致，边缘赔率越高
3. **难度影响**：难度主要影响赔率分布的极端程度
4. **对称性**：赔率分布完全对称，左右槽位赔率相同
5. **客户端种子**：通过 Seed Service API 管理，游戏自动使用已设置的种子
6. **动画建议**：客户端应实现球的下落动画以增强游戏体验
7. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成

## 7. 公平性验证 API

除了 WebSocket 接口外，系统还提供 RESTful API 用于独立验证游戏结果的公平性。

### 7.1 验证接口

**端点**: `POST /v1/fairness/plinko/verify`

**认证**: 需要 JWT Token

**请求参数**:

```json
{
  "rows": 12,
  "difficulty": "medium",
  "clientSeed": "player_seed_123",
  "serverSeed": "revealed_server_seed",
  "nonce": 1
}
```

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `rows` | number | 是 | 行数（8-16） |
| `difficulty` | string | 是 | 难度等级："low"、"medium"、"high" |
| `clientSeed` | string | 是 | 客户端种子（游戏时提供的） |
| `serverSeed` | string | 是 | 服务器种子（游戏结束后揭示的） |
| `nonce` | number | 是 | Nonce 值（≥0） |

**响应结果**:

```json
{
  "path": [0, 1, 1, 0, 1, 0, 1, 1, 0, 1, 0, 1],
  "finalSlot": 7,
  "multiplier": 5.6
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `path` | number[] | 落球路径（0=左，1=右） |
| `finalSlot` | number | 最终槽位（0 到 rows） |
| `multiplier` | number | 倍率 |

### 7.2 验证步骤

1. **获取游戏数据**：
   - 从游戏结果中获取 `provablyFair` 信息
   - 记录自己提供的 `clientSeed`
   - 记录游戏的 `rows` 和 `difficulty`

2. **调用验证接口**：
   ```bash
   curl -X POST https://dev.hicasino.xyz/v1/fairness/plinko/verify \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_JWT_TOKEN" \
     -d '{
       "rows": 12,
       "difficulty": "medium",
       "clientSeed": "your_client_seed",
       "serverSeed": "revealed_server_seed",
       "nonce": 1
     }'
   ```

3. **对比结果**：
   - 验证返回的 `path` 与游戏实际路径是否一致
   - 验证返回的 `finalSlot` 与最终槽位是否一致
   - 验证返回的 `multiplier` 与倍率是否一致
   - 如果全部一致，证明游戏结果公平

### 7.3 JavaScript 验证示例

```javascript
async function verifyPlinkoResult(gameResult, jwtToken) {
  const response = await fetch('https://dev.hicasino.xyz/v1/fairness/plinko/verify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${jwtToken}`
    },
    body: JSON.stringify({
      rows: gameResult.rows,
      difficulty: gameResult.difficulty,
      clientSeed: gameResult.provablyFair.clientSeed,
      serverSeed: gameResult.provablyFair.serverSeed,
      nonce: gameResult.provablyFair.nonce
    })
  });

  const verification = await response.json();

  // 对比路径
  const pathMatch = verification.path.every((dir, index) =>
    dir === gameResult.path[index]
  );

  // 对比最终槽位
  const slotMatch = verification.finalSlot === gameResult.finalSlot;

  // 对比倍率（允许小的浮点误差）
  const multiplierMatch = Math.abs(verification.multiplier - gameResult.multiplier) < 0.01;

  const isValid = pathMatch && slotMatch && multiplierMatch;
  console.log('验证结果:', isValid ? '✅ 公平' : '❌ 不匹配');
  return isValid;
}
```

### 7.4 验证原理

验证接口使用与游戏相同的算法生成落球路径：

```
使用 HMAC-SHA256 算法：
  1. 组合种子：seedStr = "serverSeed:clientSeed:nonce"
  2. 计算哈希：hash = HMAC-SHA256(key=serverSeed, message=seedStr)
  3. 按位提取路径：
     对于第 i 行（i = 0 到 rows-1）：
       byteIndex = i / 8
       bitIndex = i % 8
       bit = (hash[byteIndex] >> bitIndex) & 1
       path[i] = bit  // 0=左，1=右
  4. 计算最终槽位：
     finalSlot = path 中所有 1 的数量
  5. 查询倍率：
     multiplier = MultiplierConfig[difficulty][rows][finalSlot]
```

**哈希扩展**（当行数 > 256 时）：
```
extSeedStr = "seedStr:ext:extension_index"
重新计算 HMAC-SHA256 获得更多随机位
```

**路径示例**（12 行）：
```
path = [0,1,1,0,1,0,1,1,0,1,0,1]
       ← → → ← → ← → → ← → ← →
finalSlot = 7 (有 7 个 1)
```

这确保了：
- **确定性**：相同的种子组合总是产生相同的路径
- **不可预测性**：在服务器种子揭示前无法预测结果
- **可验证性**：任何人都可以独立验证结果
- **均匀分布**：每个方向有 50% 的概率，符合二项分布

## 8. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Plinko 游戏详细设计](./plinko-detailed-design-zh.md)