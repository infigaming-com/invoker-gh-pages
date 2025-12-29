# Blackjack 游戏 API

21 点：目标是让手牌点数接近 21 点但不超过，并击败庄家。支持要牌、停牌、加倍、分牌、保险等操作。

## blackjack.placeBet

开始新游戏。

```javascript
const result = await centrifuge.rpc('blackjack.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00'                   // "0" 为试玩
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00",
    "playerHands": [{
        "cards": [
            {"suit": "hearts", "rank": "A", "value": 11},
            {"suit": "spades", "rank": "K", "value": 10}
        ],
        "bestValue": 21,
        "isBlackjack": true,
        "canHit": false,
        "canStand": false,
        "canDouble": false,
        "canSplit": false
    }],
    "dealerHand": {
        "showCard": {"suit": "diamonds", "rank": "10", "value": 10},
        "isRevealed": false
    },
    "canInsure": false
}
```

## blackjack.hit

要牌。

```javascript
const result = await centrifuge.rpc('blackjack.hit', {
    roundId: '123456789012345678',
    handIndex: 0
});
```

## blackjack.stand

停牌。

```javascript
const result = await centrifuge.rpc('blackjack.stand', {
    roundId: '123456789012345678',
    handIndex: 0
});
```

## blackjack.double

加倍（只能在前 2 张牌时使用，加倍后只能再要 1 张牌）。

```javascript
const result = await centrifuge.rpc('blackjack.double', {
    roundId: '123456789012345678',
    handIndex: 0
});
```

## blackjack.split

分牌（只能对相同点数的牌使用）。

```javascript
const result = await centrifuge.rpc('blackjack.split', {
    roundId: '123456789012345678',
    handIndex: 0
});
```

## blackjack.insurance

购买保险（只在庄家明牌为 A 时可用）。

```javascript
const result = await centrifuge.rpc('blackjack.insurance', {
    roundId: '123456789012345678',
    acceptInsurance: true
});
```

## blackjack.checkActive / blackjack.resume

```javascript
// 检查活跃游戏
const active = await centrifuge.rpc('blackjack.checkActive', {});

// 恢复游戏
const state = await centrifuge.rpc('blackjack.resume', {
    roundId: '123456789012345678'
});
```

## 游戏结束响应

```json
{
    "roundId": "123456789012345678",
    "status": "finished",
    "playerHands": [{
        "cards": [...],
        "bestValue": 20,
        "status": "win",
        "payout": 20.00,
        "result": "win"
    }],
    "dealerHand": {
        "cards": [...],
        "bestValue": 17,
        "isRevealed": true
    },
    "totalPayout": 20.00,
    "finalPayout": 20.00
}
```

## 游戏规则

- **点数计算**：A=1 或 11，2-10=面值，J/Q/K=10
- **Blackjack**：首两张牌为 21 点，赔率 3:2
- **庄家规则**：软 17 停牌（S17），≤16 必须要牌
- **分牌规则**：A+A 分牌后每手只能再要 1 张牌，最多分 1 次
- **保险**：庄家明牌为 A 时可购买，金额为原注 50%，庄家有 Blackjack 时赔付 2:1

## 公平性验证

`POST /v1/fairness/blackjack/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1
}
```

返回：`{ "deck": [32, 51, 12, ...], "shuffledOrder": [...] }`
