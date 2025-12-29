# Chicken Road 游戏 API

小鸡过马路：每走一步倍率增加，走得越远赔率越高但风险也越大，可随时提现。支持四种难度。

## chickenroad.placeBet

开始新游戏。

```javascript
const result = await centrifuge.rpc('chickenroad.placeBet', {
    clientSeed: 'player_seed_123',
    amount: '10.00',                  // "0" 为试玩
    params: {
        difficulty: 'easy'            // easy/medium/hard/daredevil
    }
});
```

**响应**：

```json
{
    "roundId": "123456789012345678",
    "status": "playing",
    "betAmount": "10.00000000",
    "currentStep": 0,
    "maxSteps": 24,
    "difficulty": "easy",
    "currentMultiplier": "1.0000",
    "nextMultiplier": "1.0300",
    "nextProbability": 0.95,
    "completedSteps": []
}
```

## chickenroad.move

前进一步。

```javascript
const result = await centrifuge.rpc('chickenroad.move', {
    roundId: '123456789012345678'
});
```

**响应（生存）**：

```json
{
    "survived": true,
    "status": "playing",
    "currentStep": 1,
    "currentMultiplier": "1.0300",
    "nextMultiplier": "1.0700",
    "nextProbability": 0.963,
    "completedSteps": [0]
}
```

**响应（失败）**：

```json
{
    "survived": false,
    "status": "finished",
    "currentMultiplier": "0.0000",
    "finalPayout": "0.00000000"
}
```

## chickenroad.cashOut

提现当前赢利（至少完成一步）。

```javascript
const result = await centrifuge.rpc('chickenroad.cashOut', {
    roundId: '123456789012345678'
});
```

**响应**：

```json
{
    "status": "cashed_out",
    "payout": "15.30000000",
    "finalPayout": "15.30000000",
    "currentMultiplier": "1.5300",
    "completedSteps": [0, 1, 2, 3, 4, 5, 6, 7, 8]
}
```

## chickenroad.checkActive / chickenroad.resume

```javascript
// 检查活跃游戏
const active = await centrifuge.rpc('chickenroad.checkActive', {});

// 恢复游戏
const state = await centrifuge.rpc('chickenroad.resume', {
    roundId: '123456789012345678'
});
```

## 难度配置

| 难度 | 最大步数 | RTP | 特点 |
|------|---------|-----|------|
| Easy | 24 | 98% | 步数多，适合稳健玩家 |
| Medium | 22 | 98% | 中等步数，平衡风险 |
| Hard | 20 | 98% | 较少步数，高风险高回报 |
| Daredevil | 15 | 98% | 极限挑战，倍率增长极快 |

**倍率示例（Easy 难度）**：

| 步数 | 1 | 5 | 10 | 15 | 20 | 24 |
|-----|-----|-----|------|------|-------|-------|
| 倍率 | 1.03x | 1.29x | 2.13x | 4.82x | 11.99x | 19.44x |

## 公平性验证

`POST /v1/fairness/chickenroad/verify`

```json
{
    "clientSeed": "player_seed_123",
    "serverSeed": "revealed_server_seed",
    "nonce": 1,
    "difficulty": "easy"
}
```

返回：`{ "stepResults": [true, true, false, ...], "multipliers": [1.03, 1.08, ...] }`
