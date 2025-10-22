# DragonTower WebSocket API 文档

## 状态标记
状态：✅ 已实现

## API 概述

DragonTower 游戏通过 WebSocket 实时通信，支持下注、闯关、兑现、状态查询等操作。

## 消息格式

### 请求消息结构
```json
{
    "id": "unique-message-id",
    "type": "MESSAGE_TYPE",
    "payload": {
        // 消息特定参数
    }
}
```

### 响应消息结构
```json
{
    "id": "unique-message-id",
    "type": "RESPONSE_TYPE",
    "payload": {
        // 响应数据
    }
}
```

### 错误响应结构
```json
{
    "id": "unique-message-id",
    "type": "ERROR",
    "payload": {
        "code": "ERROR_CODE",
        "message": "错误描述"
    }
}
```

## API 接口

### 1. 下注开始游戏 (PLACE_BET)

开始一局新的 DragonTower 游戏。

**请求消息**
```json
{
    "id": "msg-001",
    "type": "PLACE_BET",
    "payload": {
        "gameId": "inhousegame:dragontower",
        "amount": "10.00",
        "gameParams": {
            "dragonTower": {
                "difficulty": "medium"
            }
        }
    }
}
```

**请求参数**
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| gameId | string | 是 | 游戏ID，固定为 "inhousegame:dragontower" |
| amount | string | 是 | 下注金额（字符串格式） |
| gameParams.dragonTower.difficulty | string | 是 | 难度：easy/medium/hard/expert/master |

**响应消息**
```json
{
    "id": "msg-001",
    "type": "PLACE_BET_RESPONSE",
    "payload": {
        "roundId": "7134567890123456",
        "status": "playing",
        "currentLayer": 0,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "1.00",
        "completedLayers": [],
        "survived": true,
        "cashedOut": false,
        "betAmount": "10.00000000",
        "nextMultiplier": "1.47",
        "nextProbability": 0.6667,
        "balance": "990.00000000"
    }
}
```

**响应字段**
| 字段 | 类型 | 说明 |
|------|------|------|
| roundId | string | 回合ID |
| status | string | 游戏状态：ready/playing/finished/cashed_out |
| currentLayer | int | 当前层数（0-9） |
| maxLayers | int | 最大层数（固定9） |
| difficulty | string | 难度档位 |
| currentMultiplier | string | 当前倍率（字符串格式，2位小数） |
| completedLayers | array | 已完成层数列表 |
| survived | bool | 是否存活 |
| cashedOut | bool | 是否已兑现 |
| betAmount | string | 下注金额 |
| nextMultiplier | string | 下一层倍率 |
| nextProbability | float | 下一层成功率 |
| balance | string | 当前余额 |

---

### 2. 闯关下一层 (DRAGONTOWER_CLIMB)

尝试闯关下一层塔楼。

**请求消息**
```json
{
    "id": "msg-002",
    "type": "DRAGONTOWER_CLIMB",
    "payload": {
        "roundId": "7134567890123456"
    }
}
```

**请求参数**
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| roundId | string | 是 | 回合ID |

**响应消息（成功）**
```json
{
    "id": "msg-002",
    "type": "DRAGONTOWER_CLIMB_RESPONSE",
    "payload": {
        "roundId": "7134567890123456",
        "survived": true,
        "status": "playing",
        "currentLayer": 1,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "1.47",
        "completedLayers": [0],
        "survived": true,
        "cashedOut": false,
        "betAmount": "10.00000000",
        "nextMultiplier": "2.21",
        "nextProbability": 0.6667
    }
}
```

**响应消息（失败 - 游戏结束）**
```json
{
    "id": "msg-002",
    "type": "DRAGONTOWER_CLIMB_RESPONSE",
    "payload": {
        "roundId": "7134567890123456",
        "survived": false,
        "status": "finished",
        "currentLayer": 1,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "0.00",
        "completedLayers": [0],
        "survived": false,
        "cashedOut": false,
        "betAmount": "10.00000000",
        "nextMultiplier": "0.00",
        "nextProbability": 0,
        "balance": "990.00000000"
    }
}
```

**响应字段**
| 字段 | 类型 | 说明 |
|------|------|------|
| survived | bool | 本层是否成功 |
| status | string | 游戏状态 |
| currentLayer | int | 当前层数 |
| currentMultiplier | string | 当前倍率 |
| balance | string | 余额（仅游戏结束时返回） |

---

### 3. 兑现 (DRAGONTOWER_CASHOUT)

主动兑现当前倍率。

**请求消息**
```json
{
    "id": "msg-003",
    "type": "DRAGONTOWER_CASHOUT",
    "payload": {
        "roundId": "7134567890123456"
    }
}
```

**请求参数**
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| roundId | string | 是 | 回合ID |

**响应消息**
```json
{
    "id": "msg-003",
    "type": "DRAGONTOWER_CASHOUT_RESPONSE",
    "payload": {
        "roundId": "7134567890123456",
        "status": "cashed_out",
        "currentLayer": 3,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "3.31",
        "completedLayers": [0, 1, 2],
        "survived": true,
        "cashedOut": true,
        "betAmount": "10.00000000",
        "payout": "33.10000000",
        "balance": "1023.10000000",
        "nextMultiplier": "3.31",
        "nextProbability": 0.6667
    }
}
```

**响应字段**
| 字段 | 类型 | 说明 |
|------|------|------|
| payout | string | 赔付金额 |
| balance | string | 当前余额 |
| status | string | 游戏状态（cashed_out） |

---

### 4. 检查活跃游戏 (DRAGONTOWER_CHECK_ACTIVE)

检查玩家是否有正在进行的游戏。

**请求消息**
```json
{
    "id": "msg-004",
    "type": "DRAGONTOWER_CHECK_ACTIVE",
    "payload": {}
}
```

**响应消息（有活跃游戏）**
```json
{
    "id": "msg-004",
    "type": "DRAGONTOWER_CHECK_ACTIVE_RESPONSE",
    "payload": {
        "hasActiveGame": true,
        "roundId": "7134567890123456",
        "status": "playing",
        "currentLayer": 2,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "2.21",
        "completedLayers": [0, 1],
        "survived": true,
        "cashedOut": false,
        "betAmount": "10.00000000",
        "nextMultiplier": "3.31",
        "nextProbability": 0.6667
    }
}
```

**响应消息（无活跃游戏）**
```json
{
    "id": "msg-004",
    "type": "DRAGONTOWER_CHECK_ACTIVE_RESPONSE",
    "payload": {
        "hasActiveGame": false
    }
}
```

---

### 5. 恢复游戏 (DRAGONTOWER_RESUME)

恢复指定回合的游戏状态（断线重连）。

**请求消息**
```json
{
    "id": "msg-005",
    "type": "DRAGONTOWER_RESUME",
    "payload": {
        "roundId": "7134567890123456"
    }
}
```

**请求参数**
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| roundId | string | 是 | 回合ID |

**响应消息**
```json
{
    "id": "msg-005",
    "type": "DRAGONTOWER_RESUME_RESPONSE",
    "payload": {
        "roundId": "7134567890123456",
        "status": "playing",
        "currentLayer": 2,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "2.21",
        "completedLayers": [0, 1],
        "survived": true,
        "cashedOut": false,
        "betAmount": "10.00000000",
        "nextMultiplier": "3.31",
        "nextProbability": 0.6667
    }
}
```

---

### 6. 获取游戏状态 (DRAGONTOWER_GET_STATE)

获取指定回合的当前游戏状态。

**请求消息**
```json
{
    "id": "msg-006",
    "type": "DRAGONTOWER_GET_STATE",
    "payload": {
        "roundId": "7134567890123456"
    }
}
```

**请求参数**
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| roundId | string | 是 | 回合ID |

**响应消息**
```json
{
    "id": "msg-006",
    "type": "DRAGONTOWER_GET_STATE_RESPONSE",
    "payload": {
        "roundId": "7134567890123456",
        "status": "playing",
        "currentLayer": 2,
        "maxLayers": 9,
        "difficulty": "medium",
        "currentMultiplier": "2.21",
        "completedLayers": [0, 1],
        "survived": true,
        "cashedOut": false,
        "betAmount": "10.00000000",
        "nextMultiplier": "3.31",
        "nextProbability": 0.6667
    }
}
```

---

## 错误码

| 错误码 | 说明 |
|--------|------|
| UNAUTHORIZED | 未授权（未登录） |
| INVALID_PAYLOAD | 无效的请求数据 |
| INVALID_GAME_PARAMS | 无效的游戏参数 |
| INVALID_AMOUNT | 无效的下注金额 |
| INVALID_DIFFICULTY | 无效的难度档位 |
| INSUFFICIENT_BALANCE | 余额不足 |
| BET_FAILED | 下注失败 |
| CLIMB_FAILED | 闯关失败 |
| CASHOUT_FAILED | 兑现失败 |
| GAME_NOT_FOUND | 游戏不存在 |
| GAME_ALREADY_ENDED | 游戏已结束 |
| GAME_NOT_STARTED | 游戏未开始 |
| ALREADY_AT_MAX_LAYER | 已达到最高层 |
| INTERNAL_ERROR | 服务器内部错误 |

## 完整游戏流程示例

### 示例1：成功闯关3层后兑现

```javascript
// 1. 下注开始游戏
send({
    id: "1",
    type: "PLACE_BET",
    payload: {
        gameId: "inhousegame:dragontower",
        amount: "10.00",
        gameParams: {
            dragonTower: { difficulty: "medium" }
        }
    }
});
// 响应：currentLayer=0, currentMultiplier=1.00, nextMultiplier=1.47

// 2. 闯关第1层（成功）
send({
    id: "2",
    type: "DRAGONTOWER_CLIMB",
    payload: { roundId: "7134567890123456" }
});
// 响应：survived=true, currentLayer=1, currentMultiplier=1.47

// 3. 闯关第2层（成功）
send({
    id: "3",
    type: "DRAGONTOWER_CLIMB",
    payload: { roundId: "7134567890123456" }
});
// 响应：survived=true, currentLayer=2, currentMultiplier=2.21

// 4. 闯关第3层（成功）
send({
    id: "4",
    type: "DRAGONTOWER_CLIMB",
    payload: { roundId: "7134567890123456" }
});
// 响应：survived=true, currentLayer=3, currentMultiplier=3.31

// 5. 兑现
send({
    id: "5",
    type: "DRAGONTOWER_CASHOUT",
    payload: { roundId: "7134567890123456" }
});
// 响应：payout=33.10, balance=1023.10
```

### 示例2：第2层失败

```javascript
// 1. 下注开始游戏
send({
    id: "1",
    type: "PLACE_BET",
    payload: {
        gameId: "inhousegame:dragontower",
        amount: "10.00",
        gameParams: {
            dragonTower: { difficulty: "hard" }
        }
    }
});

// 2. 闯关第1层（成功）
send({
    id: "2",
    type: "DRAGONTOWER_CLIMB",
    payload: { roundId: "7134567890123456" }
});
// 响应：survived=true, currentLayer=1, currentMultiplier=1.96

// 3. 闯关第2层（失败）
send({
    id: "3",
    type: "DRAGONTOWER_CLIMB",
    payload: { roundId: "7134567890123456" }
});
// 响应：survived=false, status=finished, currentMultiplier=0.00, balance=990.00
```

### 示例3：断线重连

```javascript
// 1. 检查活跃游戏
send({
    id: "1",
    type: "DRAGONTOWER_CHECK_ACTIVE",
    payload: {}
});
// 响应：hasActiveGame=true, roundId="7134567890123456", currentLayer=2

// 2. 恢复游戏状态
send({
    id: "2",
    type: "DRAGONTOWER_RESUME",
    payload: { roundId: "7134567890123456" }
});
// 响应：完整的游戏状态

// 3. 继续游戏
send({
    id: "3",
    type: "DRAGONTOWER_CLIMB",
    payload: { roundId: "7134567890123456" }
});
```

## 注意事项

1. **客户端种子**：下注前需通过 `SET_CLIENT_SEED` 消息设置客户端种子
2. **金额格式**：所有金额字段使用字符串格式，最多8位小数
3. **倍率精度**：倍率显示为2位小数（四舍五入）
4. **并发限制**：同一玩家同一时刻只能有一局进行中的游戏
5. **状态持久化**：游戏状态会自动保存，支持断线重连
6. **幂等性**：重复的操作请求会返回相同的结果，不会改变游戏状态
