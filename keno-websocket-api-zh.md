# Keno 游戏 WebSocket API 文档

## 游戏概述

Keno 是一款经典的数字彩票游戏，玩家从 1-40 中选择 1-10 个数字，系统随机开出 10 个号码。根据匹配的数字数量获得相应赔率的派彩。

## 游戏特性

- **即时开奖**：单次下注立即开奖
- **灵活选择**：可选择 1-10 个数字
- **多种难度**：支持 low、classic、medium、high 四种难度模式
- **高赔率**：最高可达 100000 倍（选10中10）
- **可证明公平**：完整的 Provably Fair 机制
- **试玩模式**：支持免费试玩体验

## 1. PLACE_BET - Keno 下注

发起 Keno 游戏下注，立即开奖。

### 请求消息

```json
{
  "i": "msg_501",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",  // 可选：空字符串""或"0"表示试玩模式
    "gameParams": {
      "keno": {
        "selectedNumbers": [3, 7, 15, 22, 28],  // 选择1-10个数字（1-40范围内）
        "difficulty": "classic"  // 可选，默认为classic
      }
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 下注金额，空字符串或"0"为试玩模式 |
| `keno.selectedNumbers` | number[] | 是 | 选择的数字，1-40范围内，1-10个不重复数字 |
| `keno.difficulty` | string | 否 | 难度模式：low、classic、medium、high |

### 难度模式说明

- **low**: 低风险低回报，适合保守玩家
- **classic**: 经典模式，平衡风险和回报（默认）
- **medium**: 中等风险，中等回报
- **high**: 高风险高回报，适合追求刺激的玩家

### 响应消息

```json
{
  "i": "msg_501",
  "t": "PLACE_BET_RESPONSE",
  "p": {
    "roundId": "123456789012345678",
    "gameResult": {
      "gameId": "inhousegame:keno",
      "betAmount": "10.00",
      "winAmount": "25.00",
      "isWin": true,
      "multiplier": "2.5",
      "timestamp": 1704067200000,
      "kenoOutcome": {
        "selectedNumbers": [3, 7, 15, 22, 28],
        "drawnNumbers": [3, 7, 9, 12, 15, 18, 22, 25, 31, 35],
        "matchedNumbers": [3, 7, 15, 22],
        "matchCount": 4,
        "spotsCount": 5,
        "multiplier": "2.5",
        "difficulty": "classic"
      }
    },
    "balance": "1025.00"
  }
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID，纯数字格式 |
| `gameResult.betAmount` | string | 下注金额 |
| `gameResult.winAmount` | string | 派彩金额（下注金额 × 赔率） |
| `gameResult.isWin` | boolean | 是否有派彩（赔率 > 0） |
| `kenoOutcome.selectedNumbers` | number[] | 玩家选择的数字 |
| `kenoOutcome.drawnNumbers` | number[] | 系统开出的10个号码 |
| `kenoOutcome.matchedNumbers` | number[] | 匹配的数字 |
| `kenoOutcome.matchCount` | number | 匹配数量 |
| `kenoOutcome.spotsCount` | number | 选择的数字数量 |
| `kenoOutcome.multiplier` | string | 赔率倍数 |
| `kenoOutcome.difficulty` | string | 使用的难度模式 |

## 2. 赔率表

### Classic 难度赔率表

| 选择数量 | 匹配0 | 匹配1 | 匹配2 | 匹配3 | 匹配4 | 匹配5 | 匹配6 | 匹配7 | 匹配8 | 匹配9 | 匹配10 |
|---------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|--------|
| 1 | 0 | 3.75 | - | - | - | - | - | - | - | - | - |
| 2 | 0 | 0 | 9 | - | - | - | - | - | - | - | - |
| 3 | 0 | 0 | 2 | 16 | - | - | - | - | - | - | - |
| 4 | 0 | 0 | 1 | 4 | 40 | - | - | - | - | - | - |
| 5 | 0 | 0 | 0 | 1 | 2.5 | 10 | - | - | - | - | - |
| 6 | 0 | 0 | 0 | 1 | 2 | 5 | 75 | - | - | - | - |
| 7 | 0 | 0 | 0 | 0.5 | 1 | 2 | 15 | 350 | - | - | - |
| 8 | 0 | 0 | 0 | 0 | 0.5 | 1 | 5 | 50 | 1000 | - | - |
| 9 | 0 | 0 | 0 | 0 | 0.5 | 1 | 2 | 20 | 200 | 4000 | - |
| 10 | 0 | 0 | 0 | 0 | 0 | 0.5 | 1 | 10 | 100 | 1000 | 10000 |

### 难度模式对比

| 难度 | RTP | 特点 | 适合人群 |
|------|-----|------|----------|
| Low | 70-75% | 小奖频繁，大奖稀少 | 保守玩家 |
| Classic | 75-80% | 平衡设计 | 普通玩家 |
| Medium | 75-80% | 中等波动 | 中级玩家 |
| High | 70-75% | 大奖诱人，小奖稀少 | 追求刺激玩家 |

## 3. 游戏规则

1. **数字选择**
   - 从 1-40 中选择 1-10 个不重复的数字
   - 选择越多，潜在赔率越高

2. **开奖机制**
   - 系统随机开出 10 个号码
   - 使用 Provably Fair 算法确保公平

3. **赔付计算**
   - 根据选择数量和匹配数量查询赔率表
   - 赔付 = 下注金额 × 赔率

4. **派彩条件**
   - 根据匹配数量查询赔率表获得相应派彩
   - 不同选择数量和匹配数量对应不同的赔率

## 4. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `INVALID_NUMBER_COUNT` | 选择的数字数量不在1-10范围内 | 调整选择数量 |
| `INVALID_NUMBER_RANGE` | 选择的数字不在1-40范围内 | 选择1-40之间的数字 |
| `DUPLICATE_NUMBERS` | 选择的数字有重复 | 确保数字不重复 |
| `INVALID_DIFFICULTY` | 无效的难度模式 | 使用有效的难度值 |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 通过 Seed Service API 设置客户端种子 |
| `INSUFFICIENT_BALANCE` | 余额不足 | 充值或减少下注金额 |

## 5. 注意事项

1. **即时游戏**：Keno 是即时开奖游戏，无需会话管理
2. **数字范围**：必须选择 1-40 范围内的数字
3. **选择数量**：可选择 1-10 个数字，不同数量有不同赔率表
4. **难度选择**：不同难度影响赔率分布，选择适合自己的模式
5. **客户端种子**：通过 Seed Service API 管理，游戏自动使用已设置的种子
6. **快速选号**：快速选号功能由客户端实现，服务端只验证合法性
7. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成

## 6. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Keno 游戏详细设计](./keno-detailed-design-zh.md)