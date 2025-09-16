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

## 6. 游戏配置获取

### GET_GAME_CONFIG 请求

获取 Keno 游戏的完整配置，包括所有难度的赔率表。

```json
{
  "i": "msg_config_001",
  "t": "GET_GAME_CONFIG",
  "p": {
    "gameId": "inhousegame:keno"
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
        "gameId": "inhousegame:keno",
        "config": {
          "id": 2000005,
          "gameName": "Keno",
          "gameId": "inhousegame:keno",
          "category": "instant",
          "status": "active",
          "description": "Classic lottery-style game with 80 numbers",
          "minBet": 0.1,
          "maxBet": 1000,
          "defaultRTP": "97%",
          "features": ["provably_fair", "instant_play", "quick_pick", "multi_spot"],
          "betInfo": [
            {
              "currency": "USD",
              "currencyType": "fiat",
              "defaultBet": 1,
              "minBet": 0.1,
              "maxBet": 1000,
              "maxProfit": 100000
            }
          ],
          "gameParameters": {
            "numberRange": {
              "min": 1,
              "max": 40
            },
            "spotRange": {
              "min": 1,
              "max": 10
            },
            "drawnNumbers": 10,
            "payoutTables": [
              {
                "difficulty": "low",
                "payouts": [
                  {"spots": 1, "entries": [{"hits": 0, "payout": 0.7}, {"hits": 1, "payout": 1.85}]},
                  {"spots": 2, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 3.8}]},
                  {"spots": 3, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 1.1}, {"hits": 2, "payout": 1.38}, {"hits": 3, "payout": 26}]},
                  {"spots": 4, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 2.2}, {"hits": 3, "payout": 7.9}, {"hits": 4, "payout": 90}]},
                  {"spots": 5, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.5}, {"hits": 3, "payout": 4.2}, {"hits": 4, "payout": 13}, {"hits": 5, "payout": 300}]},
                  {"spots": 6, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.1}, {"hits": 3, "payout": 2}, {"hits": 4, "payout": 6.2}, {"hits": 5, "payout": 100}, {"hits": 6, "payout": 700}]},
                  {"spots": 7, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.1}, {"hits": 3, "payout": 1.6}, {"hits": 4, "payout": 3.5}, {"hits": 5, "payout": 15}, {"hits": 6, "payout": 225}, {"hits": 7, "payout": 700}]},
                  {"spots": 8, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.1}, {"hits": 3, "payout": 1.5}, {"hits": 4, "payout": 2}, {"hits": 5, "payout": 5.5}, {"hits": 6, "payout": 39}, {"hits": 7, "payout": 100}, {"hits": 8, "payout": 800}]},
                  {"spots": 9, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.1}, {"hits": 3, "payout": 1.3}, {"hits": 4, "payout": 1.7}, {"hits": 5, "payout": 2.5}, {"hits": 6, "payout": 7.5}, {"hits": 7, "payout": 50}, {"hits": 8, "payout": 250}, {"hits": 9, "payout": 1000}]},
                  {"spots": 10, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.1}, {"hits": 3, "payout": 1.2}, {"hits": 4, "payout": 1.3}, {"hits": 5, "payout": 1.8}, {"hits": 6, "payout": 3.5}, {"hits": 7, "payout": 13}, {"hits": 8, "payout": 50}, {"hits": 9, "payout": 250}, {"hits": 10, "payout": 1000}]}
                ]
              },
              {
                "difficulty": "classic",
                "payouts": [
                  {"spots": 1, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 3.96}]},
                  {"spots": 2, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 1.9}, {"hits": 2, "payout": 4.5}]},
                  {"spots": 3, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 1}, {"hits": 2, "payout": 3.1}, {"hits": 3, "payout": 10.4}]},
                  {"spots": 4, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 0.8}, {"hits": 2, "payout": 1.8}, {"hits": 3, "payout": 5}, {"hits": 4, "payout": 22.5}]},
                  {"spots": 5, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 0.25}, {"hits": 2, "payout": 1.4}, {"hits": 3, "payout": 4.1}, {"hits": 4, "payout": 16.5}, {"hits": 5, "payout": 36}]},
                  {"spots": 6, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1}, {"hits": 3, "payout": 3.68}, {"hits": 4, "payout": 7}, {"hits": 5, "payout": 16.5}, {"hits": 6, "payout": 40}]},
                  {"spots": 7, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 0.47}, {"hits": 3, "payout": 3}, {"hits": 4, "payout": 4.5}, {"hits": 5, "payout": 14}, {"hits": 6, "payout": 31}, {"hits": 7, "payout": 60}]},
                  {"spots": 8, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 2.2}, {"hits": 4, "payout": 4}, {"hits": 5, "payout": 13}, {"hits": 6, "payout": 22}, {"hits": 7, "payout": 55}, {"hits": 8, "payout": 70}]},
                  {"spots": 9, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 1.55}, {"hits": 4, "payout": 3}, {"hits": 5, "payout": 8}, {"hits": 6, "payout": 15}, {"hits": 7, "payout": 44}, {"hits": 8, "payout": 60}, {"hits": 9, "payout": 85}]},
                  {"spots": 10, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 1.4}, {"hits": 4, "payout": 2.25}, {"hits": 5, "payout": 4.5}, {"hits": 6, "payout": 8}, {"hits": 7, "payout": 17}, {"hits": 8, "payout": 50}, {"hits": 9, "payout": 80}, {"hits": 10, "payout": 100}]}
                ]
              },
              {
                "difficulty": "medium",
                "payouts": [
                  {"spots": 1, "entries": [{"hits": 0, "payout": 0.4}, {"hits": 1, "payout": 2.75}]},
                  {"spots": 2, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 1.8}, {"hits": 2, "payout": 5.1}]},
                  {"spots": 3, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 2.8}, {"hits": 3, "payout": 50}]},
                  {"spots": 4, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.7}, {"hits": 3, "payout": 10}, {"hits": 4, "payout": 100}]},
                  {"spots": 5, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 1.4}, {"hits": 3, "payout": 4}, {"hits": 4, "payout": 14}, {"hits": 5, "payout": 390}]},
                  {"spots": 6, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 3}, {"hits": 4, "payout": 9}, {"hits": 5, "payout": 180}, {"hits": 6, "payout": 710}]},
                  {"spots": 7, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 2}, {"hits": 4, "payout": 7}, {"hits": 5, "payout": 30}, {"hits": 6, "payout": 400}, {"hits": 7, "payout": 800}]},
                  {"spots": 8, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 2}, {"hits": 4, "payout": 4}, {"hits": 5, "payout": 11}, {"hits": 6, "payout": 67}, {"hits": 7, "payout": 400}, {"hits": 8, "payout": 900}]},
                  {"spots": 9, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 2}, {"hits": 4, "payout": 2.5}, {"hits": 5, "payout": 5}, {"hits": 6, "payout": 15}, {"hits": 7, "payout": 100}, {"hits": 8, "payout": 500}, {"hits": 9, "payout": 1000}]},
                  {"spots": 10, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 1.6}, {"hits": 4, "payout": 2}, {"hits": 5, "payout": 4}, {"hits": 6, "payout": 7}, {"hits": 7, "payout": 26}, {"hits": 8, "payout": 100}, {"hits": 9, "payout": 500}, {"hits": 10, "payout": 1000}]}
                ]
              },
              {
                "difficulty": "high",
                "payouts": [
                  {"spots": 1, "entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 3.96}]},
                  {"spots": 2, "entries": [{"hits": 0, "payout": 0}, {"hits": 2, "payout": 17.1}]},
                  {"spots": 3, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 81.5}]},
                  {"spots": 4, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 10}, {"hits": 4, "payout": 259}]},
                  {"spots": 5, "entries": [{"hits": 0, "payout": 0}, {"hits": 3, "payout": 4.5}, {"hits": 4, "payout": 48}, {"hits": 5, "payout": 450}]},
                  {"spots": 6, "entries": [{"hits": 0, "payout": 0}, {"hits": 4, "payout": 11}, {"hits": 5, "payout": 350}, {"hits": 6, "payout": 710}]},
                  {"spots": 7, "entries": [{"hits": 0, "payout": 0}, {"hits": 4, "payout": 7}, {"hits": 5, "payout": 90}, {"hits": 6, "payout": 400}, {"hits": 7, "payout": 800}]},
                  {"spots": 8, "entries": [{"hits": 0, "payout": 0}, {"hits": 4, "payout": 5}, {"hits": 5, "payout": 20}, {"hits": 6, "payout": 270}, {"hits": 7, "payout": 600}, {"hits": 8, "payout": 900}]},
                  {"spots": 9, "entries": [{"hits": 0, "payout": 0}, {"hits": 4, "payout": 4}, {"hits": 5, "payout": 11}, {"hits": 6, "payout": 56}, {"hits": 7, "payout": 500}, {"hits": 8, "payout": 800}, {"hits": 9, "payout": 1000}]},
                  {"spots": 10, "entries": [{"hits": 0, "payout": 0}, {"hits": 4, "payout": 3.5}, {"hits": 5, "payout": 8}, {"hits": 6, "payout": 13}, {"hits": 7, "payout": 63}, {"hits": 8, "payout": 500}, {"hits": 9, "payout": 800}, {"hits": 10, "payout": 1000}]}
                ]
              }
            ]
          },
          "rtpConfig": {
            "displayRTP": [97, 96, 95, 94],
            "actualRTP": [97, 96, 95, 94, 93, 92, 90]
          },
          "commissionRate": "1%",
          "maxRewardMultiplier": 50000
        }
      }
    ]
  }
}
```

### 配置响应说明

GET_GAME_CONFIG 响应中包含完整的游戏配置，关键字段说明：

#### payoutTables 结构
`payoutTables` 字段包含所有4个难度的完整赔率表：

- **low**: 低风险模式赔率 - 小奖频繁，大奖稀少
- **classic**: 经典模式赔率（默认）- 平衡设计
- **medium**: 中等风险模式赔率 - 中等波动
- **high**: 高风险模式赔率 - 大奖诱人，小奖稀少

每个难度包含：
- `difficulty`: 难度名称
- `payouts`: 包含 spots 1-10 的赔率配置数组
  - `spots`: 选择的数字数量
  - `entries`: 该选择数量下的赔率列表
    - `hits`: 命中数量
    - `payout`: 对应的赔率倍数

#### 其他重要字段
- `numberRange`: 可选数字范围（1-40）
- `spotRange`: 可选择的数字数量范围（1-10）
- `drawnNumbers`: 每次开出的数字数量（10个）
- `betInfo`: 各币种的投注限额
- `rtpConfig`: RTP（返还率）配置
- `maxRewardMultiplier`: 最大奖励倍数

前端应根据玩家选择的难度，从对应的难度配置中获取赔率显示。

## 7. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Keno 游戏详细设计](./keno-detailed-design-zh.md)