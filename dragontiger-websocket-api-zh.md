# Dragon Tiger 游戏 WebSocket API 文档

## 游戏概述

Dragon Tiger（龙虎斗）是一款经典的亚洲卡牌游戏，玩家预测龙或虎谁的牌点数更大，或者预测平局。游戏简单快速，支持三种投注类型，RTP 为 97.1%。

## 游戏特性

- **即时游戏**：单次下注立即开奖
- **可证明公平**：支持完整的 Provably Fair 机制
- **三种投注**：龙投注、虎投注、和投注
- **试玩模式**：支持免费试玩体验
- **佣金机制**：龙/虎投注胜利扣除 5% 佣金
- **平局退款**：平局时龙/虎投注退回 50%

## 1. PLACE_BET - 下注

发起 Dragon Tiger 游戏下注。支持试玩模式（amount为空或"0"）。

### 请求消息

```json
{
  "i": "msg_007",
  "t": "PLACE_BET",
  "p": {
    "amount": "10.00",  // 可选：空字符串""或"0"表示试玩模式
    "gameParams": {
      "dragonTiger": {
        "dragonBet": "10.00",  // 龙投注金额
        "tigerBet": "0",       // 虎投注金额
        "tieBet": "0"          // 和投注金额
      }
    }
  }
}
```

### 参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `amount` | string | 是 | 总投注金额（dragonBet + tigerBet + tieBet），空字符串或"0"为试玩模式 |
| `gameParams.dragonTiger.dragonBet` | string | 是 | 龙投注金额，非负数 |
| `gameParams.dragonTiger.tigerBet` | string | 是 | 虎投注金额，非负数 |
| `gameParams.dragonTiger.tieBet` | string | 是 | 和投注金额，非负数 |

### 投注限制

- 龙和虎不能同时投注（dragonBet 和 tigerBet 不能同时大于 0）
- 总投注金额必须大于 0（至少一种投注类型有金额）
- 所有金额必须为非负数

### 响应消息

```json
{
  "i": "msg_007",
  "t": "PLACE_BET",
  "p": {
    "roundId": "123456789012345678",  // 纯数字的回合ID
    "gameResult": {
      "gameId": "inhousegame:dragontiger",
      "betAmount": "10.00",
      "winAmount": "9.50",
      "isWin": true,
      "dragonTigerOutcome": {
        "dragonCard": 10,      // 龙的牌点数 (1-13)
        "tigerCard": 5,        // 虎的牌点数 (1-13)
        "result": "dragon",    // 结果："dragon", "tiger", 或 "tie"
        "dragonPayout": "9.50000000",  // 龙投注派彩
        "tigerPayout": "0.00000000",   // 虎投注派彩
        "tiePayout": "0.00000000"      // 和投注派彩
      },
      "multiplier": "0.95",
      "timestamp": 1704067200000
    },
    "balance": "1009.50"
  }
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `roundId` | string | 游戏回合ID，纯数字格式 |
| `gameResult.betAmount` | string | 总投注金额 |
| `gameResult.winAmount` | string | 总获胜金额（所有派彩之和） |
| `gameResult.isWin` | boolean | 是否获胜 |
| `dragonTigerOutcome.dragonCard` | number | 龙的牌点数，1-13 对应 A-K |
| `dragonTigerOutcome.tigerCard` | number | 虎的牌点数，1-13 对应 A-K |
| `dragonTigerOutcome.result` | string | 游戏结果："dragon"（龙赢）、"tiger"（虎赢）、"tie"（平局） |
| `dragonTigerOutcome.dragonPayout` | string | 龙投注派彩（8位小数） |
| `dragonTigerOutcome.tigerPayout` | string | 虎投注派彩（8位小数） |
| `dragonTigerOutcome.tiePayout` | string | 和投注派彩（8位小数） |
| `gameResult.multiplier` | string | 总倍率（总派彩 / 总投注） |
| `balance` | string | 更新后的余额 |

## 2. GET_BET_HISTORY - 获取投注历史

获取玩家的 Dragon Tiger 游戏投注历史记录。

### 请求消息

```json
{
  "i": "msg_008",
  "t": "GET_BET_HISTORY",
  "p": {
    "gameId": "inhousegame:dragontiger",  // 可选，指定游戏类型
    "limit": 10,                           // 可选，默认10
    "offset": 0                            // 可选，默认0
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
        "gameId": "inhousegame:dragontiger",
        "betAmount": "10.00",
        "winAmount": "9.50",
        "isWin": true,
        "multiplier": "0.95",
        "dragonTigerOutcome": {
          "dragonCard": 10,
          "tigerCard": 5,
          "result": "dragon",
          "dragonPayout": "9.50000000",
          "tigerPayout": "0.00000000",
          "tiePayout": "0.00000000"
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

### 卡牌点数

- 卡牌范围：1-13
- 1 = A（Ace，最小）
- 2-10 = 数字牌
- 11 = J（Jack）
- 12 = Q（Queen）
- 13 = K（King，最大）

### 赔率计算

#### 龙投注 / 虎投注
- **基础赔率**：1:1
- **实际赔率**：扣除 5% 佣金后为 0.95:1
- **示例**：投注 10.00，赢得 9.50（10 × 0.95）

#### 和投注
- **赔率**：1:8
- **无佣金**：和投注不扣除佣金
- **示例**：投注 10.00，赢得 80.00（10 × 8）

#### 平局退款
- 当结果为平局（tie）时：
  - 和投注按 1:8 派彩
  - 龙投注退回 50%（10.00 投注退回 5.00）
  - 虎投注退回 50%（10.00 投注退回 5.00）

### 输赢判定

1. **龙赢**：`dragonCard > tigerCard`
   - 龙投注派彩：投注额 × 0.95
   - 虎投注损失全部
   - 和投注损失全部

2. **虎赢**：`tigerCard > dragonCard`
   - 虎投注派彩：投注额 × 0.95
   - 龙投注损失全部
   - 和投注损失全部

3. **平局**：`dragonCard == tigerCard`
   - 和投注派彩：投注额 × 8
   - 龙投注退回 50%
   - 虎投注退回 50%

### 投注示例

#### 示例 1：单独龙投注
```json
{
  "dragonBet": "10.00",
  "tigerBet": "0",
  "tieBet": "0"
}
```
- 龙赢：赢得 9.50（RTP 95%）
- 虎赢：损失 10.00
- 平局：退回 5.00

#### 示例 2：单独和投注
```json
{
  "dragonBet": "0",
  "tigerBet": "0",
  "tieBet": "10.00"
}
```
- 龙赢/虎赢：损失 10.00
- 平局：赢得 80.00

#### 示例 3：龙 + 和组合投注
```json
{
  "dragonBet": "10.00",
  "tigerBet": "0",
  "tieBet": "5.00"
}
```
- 龙赢：赢得 9.50（龙派彩）
- 虎赢：损失 15.00
- 平局：赢得 45.00（和派彩 40.00 + 龙退款 5.00）

## 4. 错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `INVALID_BET_AMOUNT` | 投注金额无效 | 确保金额非负且总额大于 0 |
| `INVALID_BET_COMBINATION` | 无效的投注组合 | 龙和虎不能同时投注 |
| `INVALID_CLIENT_SEED` | 客户端种子无效 | 通过 Seed Service API 设置客户端种子 |
| `INSUFFICIENT_BALANCE` | 余额不足 | 充值或减少下注金额 |
| `BET_LIMIT_EXCEEDED` | 超出投注限额 | 调整下注金额 |
| `INVALID_GAME_PARAMS` | 游戏参数无效 | 检查 dragonTiger 参数是否正确 |

## 5. 注意事项

1. **投注限制**：龙和虎不能同时投注，这是游戏规则的一部分
2. **客户端种子**：通过 Seed Service API 管理，游戏自动使用已设置的种子
3. **试玩模式**：将 amount 设为 "0" 或空字符串即可免费试玩
4. **精度处理**：所有金额使用字符串格式，保留 8 位小数
5. **RoundID格式**：使用纯数字格式，由 Sony Flake ID 生成器生成
6. **佣金配置**：佣金率可在服务端配置，默认为 5%
7. **平局概率**：由于卡牌范围是 1-13，平局概率约为 1/13（7.69%）

## 6. 可证明公平

Dragon Tiger 游戏使用可证明公平机制，确保游戏结果的随机性和公正性：

### 种子生成
- **服务端种子**：由服务器生成，哈希值公开
- **客户端种子**：由玩家设置
- **Nonce**：每局游戏递增

### 卡牌生成算法
```
龙卡种子 = SHA256(serverSeed:clientSeed:nonce:0)
虎卡种子 = SHA256(serverSeed:clientSeed:nonce:1)
```

每个种子通过哈希函数转换为 1-13 的卡牌点数。

### 验证方法
玩家可以通过以下信息验证游戏结果：
- 服务端种子（游戏结束后公开）
- 客户端种子
- Nonce
- 游戏结果

使用相同的算法重新计算，应得到相同的卡牌结果。

## 7. 相关文档

- [WebSocket 通用接口](./common-websocket-api-zh.md)
- [Dragon Tiger 游戏详细设计](./dragontiger-detailed-design-zh.md)
