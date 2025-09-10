# Blackjack WebSocket API 文档

## 1. 游戏标识

- **游戏ID**: `inhousegame:blackjack`
- **游戏类型**: 会话型游戏（Session Game）

## 2. 通用消息格式

### 2.1 请求消息结构
```json
{
  "id": "msg_123",
  "type": "MESSAGE_TYPE",
  "payload": {
    // 消息内容
  }
}
```

### 2.2 响应消息结构
```json
{
  "id": "msg_123",
  "type": "RESPONSE_TYPE",
  "payload": {
    // 响应内容
  }
}
```

## 3. 游戏消息

### 3.1 开始游戏

#### 请求：PLACE_BET
```json
{
  "id": "msg_123",
  "type": "PLACE_BET",
  "payload": {
    "gameType": "inhousegame:blackjack",
    "amount": "10.00",
    "currency": "USD",
    "gameParams": {
      "blackjack": {
        "action": "start"
      }
    }
  }
}
```

#### 响应：BLACKJACK_GAME_STATE
```json
{
  "id": "msg_123",
  "type": "BLACKJACK_GAME_STATE",
  "payload": {
    "roundId": "123456789",
    "status": "playing",
    "betAmount": "10.00",
    "currency": "USD",
    "playerHands": [
      {
        "handIndex": 0,
        "cards": [
          {"suit": "hearts", "rank": "A", "value": 11},
          {"suit": "spades", "rank": "K", "value": 10}
        ],
        "status": "blackjack",
        "softValue": 21,
        "hardValue": 21,
        "bestValue": 21,
        "isBlackjack": true,
        "canHit": false,
        "canStand": false,
        "canDouble": false,
        "canSplit": false
      }
    ],
    "dealerHand": {
      "showCard": {"suit": "diamonds", "rank": "10", "value": 10},
      "cardCount": 2,
      "showValue": 10,
      "hasBlackjack": false,
      "isRevealed": false
    },
    "canInsure": false,
    "insurance": null,
    "activeHandIndex": 0,
    "createdAt": 1700000000000
  }
}
```

### 3.2 要牌（Hit）

#### 请求：PLACE_BET / BLACKJACK_HIT
```json
{
  "id": "msg_124",
  "type": "PLACE_BET",
  "payload": {
    "gameType": "inhousegame:blackjack",
    "gameParams": {
      "blackjack": {
        "action": "hit",
        "roundId": "123456789",
        "handIndex": 0
      }
    }
  }
}
```

#### 响应：BLACKJACK_GAME_STATE
```json
{
  "id": "msg_124",
  "type": "BLACKJACK_GAME_STATE",
  "payload": {
    "roundId": "123456789",
    "status": "playing",
    "playerHands": [
      {
        "handIndex": 0,
        "cards": [
          {"suit": "hearts", "rank": "7", "value": 7},
          {"suit": "spades", "rank": "8", "value": 8},
          {"suit": "clubs", "rank": "5", "value": 5}
        ],
        "status": "active",
        "softValue": 20,
        "hardValue": 20,
        "bestValue": 20,
        "canHit": true,
        "canStand": true,
        "canDouble": false,
        "canSplit": false
      }
    ],
    "dealerHand": {
      "showCard": {"suit": "diamonds", "rank": "10", "value": 10},
      "cardCount": 2,
      "showValue": 10,
      "isRevealed": false
    },
    "activeHandIndex": 0
  }
}
```

### 3.3 停牌（Stand）

#### 请求：PLACE_BET / BLACKJACK_STAND
```json
{
  "id": "msg_125",
  "type": "PLACE_BET",
  "payload": {
    "gameType": "inhousegame:blackjack",
    "gameParams": {
      "blackjack": {
        "action": "stand",
        "roundId": "123456789",
        "handIndex": 0
      }
    }
  }
}
```

#### 响应：BLACKJACK_GAME_STATE + BLACKJACK_DEALER_REVEAL
```json
{
  "id": "msg_125",
  "type": "BLACKJACK_DEALER_REVEAL",
  "payload": {
    "roundId": "123456789",
    "dealerHand": {
      "cards": [
        {"suit": "diamonds", "rank": "10", "value": 10},
        {"suit": "hearts", "rank": "7", "value": 7}
      ],
      "softValue": 17,
      "hardValue": 17,
      "bestValue": 17,
      "status": "stand",
      "isRevealed": true
    }
  }
}
```

### 3.4 加倍（Double Down）

#### 请求：PLACE_BET / BLACKJACK_DOUBLE
```json
{
  "id": "msg_126",
  "type": "PLACE_BET",
  "payload": {
    "gameType": "inhousegame:blackjack",
    "gameParams": {
      "blackjack": {
        "action": "double",
        "roundId": "123456789",
        "handIndex": 0
      }
    }
  }
}
```

#### 响应：BLACKJACK_GAME_STATE
```json
{
  "id": "msg_126",
  "type": "BLACKJACK_GAME_STATE",
  "payload": {
    "roundId": "123456789",
    "status": "dealer",
    "playerHands": [
      {
        "handIndex": 0,
        "cards": [
          {"suit": "hearts", "rank": "6", "value": 6},
          {"suit": "spades", "rank": "5", "value": 5},
          {"suit": "clubs", "rank": "9", "value": 9}
        ],
        "status": "stand",
        "bestValue": 20,
        "betAmount": "20.00",
        "isDoubled": true
      }
    ]
  }
}
```

### 3.5 分牌（Split）

#### 请求：PLACE_BET / BLACKJACK_SPLIT
```json
{
  "id": "msg_127",
  "type": "PLACE_BET",
  "payload": {
    "gameType": "inhousegame:blackjack",
    "gameParams": {
      "blackjack": {
        "action": "split",
        "roundId": "123456789",
        "handIndex": 0
      }
    }
  }
}
```

#### 响应：BLACKJACK_GAME_STATE
```json
{
  "id": "msg_127",
  "type": "BLACKJACK_GAME_STATE",
  "payload": {
    "roundId": "123456789",
    "status": "playing",
    "playerHands": [
      {
        "handIndex": 0,
        "cards": [
          {"suit": "hearts", "rank": "8", "value": 8},
          {"suit": "diamonds", "rank": "3", "value": 3}
        ],
        "status": "active",
        "bestValue": 11,
        "betAmount": "10.00",
        "isSplit": true,
        "canHit": true,
        "canStand": true,
        "canDouble": true,
        "canSplit": false
      },
      {
        "handIndex": 1,
        "cards": [
          {"suit": "spades", "rank": "8", "value": 8},
          {"suit": "clubs", "rank": "2", "value": 2}
        ],
        "status": "waiting",
        "bestValue": 10,
        "betAmount": "10.00",
        "isSplit": true
      }
    ],
    "activeHandIndex": 0,
    "totalBet": "20.00"
  }
}
```

### 3.6 保险（Insurance）

#### 请求：PLACE_BET / BLACKJACK_INSURANCE
```json
{
  "id": "msg_128",
  "type": "PLACE_BET",
  "payload": {
    "gameType": "inhousegame:blackjack",
    "gameParams": {
      "blackjack": {
        "action": "insurance",
        "roundId": "123456789",
        "acceptInsurance": true
      }
    }
  }
}
```

#### 响应：BLACKJACK_GAME_STATE
```json
{
  "id": "msg_128",
  "type": "BLACKJACK_GAME_STATE",
  "payload": {
    "roundId": "123456789",
    "status": "playing",
    "insurance": {
      "amount": "5.00",
      "status": "pending"
    },
    "canInsure": false,
    "playerHands": [
      {
        "handIndex": 0,
        "cards": [
          {"suit": "hearts", "rank": "Q", "value": 10},
          {"suit": "spades", "rank": "9", "value": 9}
        ],
        "status": "active",
        "bestValue": 19,
        "canHit": true,
        "canStand": true,
        "canDouble": false,
        "canSplit": false
      }
    ],
    "dealerHand": {
      "showCard": {"suit": "hearts", "rank": "A", "value": 11},
      "cardCount": 2,
      "showValue": 11,
      "isRevealed": false
    }
  }
}
```

### 3.7 游戏结果

#### 响应：BLACKJACK_GAME_RESULT
```json
{
  "id": "msg_129",
  "type": "BLACKJACK_GAME_RESULT",
  "payload": {
    "roundId": "123456789",
    "status": "finished",
    "playerHands": [
      {
        "handIndex": 0,
        "cards": [
          {"suit": "hearts", "rank": "K", "value": 10},
          {"suit": "spades", "rank": "Q", "value": 10}
        ],
        "status": "win",
        "bestValue": 20,
        "betAmount": "10.00",
        "payout": "10.00",
        "result": "win"
      }
    ],
    "dealerHand": {
      "cards": [
        {"suit": "diamonds", "rank": "10", "value": 10},
        {"suit": "hearts", "rank": "7", "value": 7}
      ],
      "bestValue": 17,
      "status": "stand",
      "isRevealed": true
    },
    "insurance": null,
    "totalBet": "10.00",
    "totalPayout": "20.00",
    "netProfit": "10.00",
    "provablyFair": {
      "clientSeed": "user_seed_123",
      "serverSeedHash": "hash_abc123",
      "nonce": 1
    },
    "completedAt": 1700000100000
  }
}
```

### 3.8 获取游戏状态

#### 请求：BLACKJACK_GET_STATE
```json
{
  "id": "msg_130",
  "type": "BLACKJACK_GET_STATE",
  "payload": {
    "roundId": "123456789"
  }
}
```

#### 响应：BLACKJACK_GAME_STATE
返回当前完整游戏状态（格式同上）

### 3.9 检查活跃游戏

#### 请求：BLACKJACK_CHECK_ACTIVE
```json
{
  "id": "msg_131",
  "type": "BLACKJACK_CHECK_ACTIVE",
  "payload": {}
}
```

#### 响应：BLACKJACK_CHECK_ACTIVE_RESPONSE
```json
{
  "id": "msg_131",
  "type": "BLACKJACK_CHECK_ACTIVE_RESPONSE",
  "payload": {
    "hasActiveGame": true,
    "roundId": "123456789",
    "gameState": {
      // 完整游戏状态
    }
  }
}
```

## 4. 错误响应

### 4.1 错误消息格式
```json
{
  "id": "msg_123",
  "type": "ERROR",
  "payload": {
    "code": "ERROR_CODE",
    "message": "错误描述"
  }
}
```

### 4.2 错误码列表

| 错误码 | 描述 |
|--------|------|
| INVALID_REQUEST | 无效请求 |
| INVALID_AMOUNT | 无效金额 |
| INSUFFICIENT_BALANCE | 余额不足 |
| INVALID_CLIENT_SEED | 无效客户端种子 |
| GAME_NOT_FOUND | 游戏不存在 |
| INVALID_ACTION | 无效操作 |
| ACTION_NOT_ALLOWED | 操作不允许 |
| HAND_NOT_FOUND | 手牌不存在 |
| ALREADY_SPLIT | 已经分牌 |
| CANNOT_SPLIT | 不能分牌 |
| CANNOT_DOUBLE | 不能加倍 |
| CANNOT_INSURE | 不能购买保险 |
| SESSION_EXPIRED | 会话过期 |
| INTERNAL_ERROR | 内部错误 |

## 5. 游戏状态说明

### 5.1 游戏整体状态
- `betting` - 下注中
- `dealing` - 发牌中
- `playing` - 玩家操作中
- `dealer` - 庄家操作中
- `finished` - 游戏结束
- `cashed_out` - 已结算

### 5.2 手牌状态
- `active` - 可操作
- `waiting` - 等待操作（多手牌时）
- `stand` - 已停牌
- `busted` - 爆牌
- `blackjack` - 天生21点
- `win` - 获胜
- `lose` - 失败
- `push` - 平局

### 5.3 保险状态
- `pending` - 待结算
- `won` - 保险赢
- `lost` - 保险输

## 6. 数据类型说明

### 6.1 牌面数据
```typescript
interface Card {
  suit: "hearts" | "diamonds" | "clubs" | "spades";  // 花色
  rank: string;  // 牌面：2-10, J, Q, K, A
  value: number; // 点数值
}
```

### 6.2 手牌数据
```typescript
interface PlayerHand {
  handIndex: number;      // 手牌索引
  cards: Card[];         // 牌列表
  status: string;        // 手牌状态
  softValue: number;     // 软牌点数
  hardValue: number;     // 硬牌点数
  bestValue: number;     // 最佳点数
  betAmount: string;     // 投注金额
  isDoubled?: boolean;   // 是否已加倍
  isSplit?: boolean;     // 是否由分牌产生
  isFromAces?: boolean;  // 是否由A+A分牌产生
  isBlackjack?: boolean; // 是否天生21点
  canHit: boolean;       // 可否要牌
  canStand: boolean;     // 可否停牌
  canDouble: boolean;    // 可否加倍
  canSplit: boolean;     // 可否分牌
  payout?: string;       // 赔付金额
  result?: string;       // 结果：win/lose/push
}
```

### 6.3 庄家手牌数据
```typescript
interface DealerHand {
  showCard?: Card;       // 明牌（游戏中）
  cards?: Card[];        // 所有牌（游戏结束）
  cardCount: number;     // 牌数量
  showValue: number;     // 明牌点数
  softValue?: number;    // 软牌点数
  hardValue?: number;    // 硬牌点数
  bestValue?: number;    // 最佳点数
  status?: string;       // 状态
  hasBlackjack: boolean; // 是否有Blackjack
  isRevealed: boolean;   // 暗牌是否已翻开
}
```

### 6.4 保险数据
```typescript
interface Insurance {
  amount: string;        // 保险金额
  status: string;        // 状态：pending/won/lost
  payout?: string;       // 赔付金额
}
```

## 7. 特殊规则说明

### 7.1 分牌规则
- 只能对相同点数的牌进行分牌
- A+A分牌后每手只能再要1张牌
- 分牌后的A+10不算Blackjack
- 最多分牌1次（最多2手牌）

### 7.2 加倍规则
- 只能在前2张牌时加倍
- 加倍后只能再要1张牌
- 分牌后的手牌可以单独加倍

### 7.3 保险规则
- 只在庄家明牌为A时提供
- 保险金额为原注的50%
- 庄家有Blackjack时赔付2:1

### 7.4 庄家规则
- 软17（A+6）必须停牌（S17规则）
- 点数≤16必须要牌
- 点数≥17必须停牌

## 8. 断线重连

### 8.1 恢复游戏
```json
{
  "id": "msg_132",
  "type": "BLACKJACK_RESUME_GAME",
  "payload": {
    "roundId": "123456789"
  }
}
```

### 8.2 响应
返回完整游戏状态，允许继续游戏

## 9. 实现状态

- ⚠️ 开始游戏
- ⚠️ 要牌操作
- ⚠️ 停牌操作
- ⚠️ 加倍操作
- ⚠️ 分牌操作
- ⚠️ 保险机制
- ⚠️ 庄家自动操作
- ⚠️ 游戏结算
- ⚠️ 断线重连
- ⚠️ 状态查询