# Limbo 游戏详细设计文档

## 状态：✅ 已实现

## 1. 游戏概述

Limbo（倍率点亮）是一款单人即时投注游戏，玩家在下注前设定目标倍率，系统生成随机结果倍率，如果结果倍率大于等于目标倍率则获胜。

### 1.1 核心特性

- **游戏类型**：单人即时结算游戏
- **下注范围**：0.000100 - 500,000 USD（支持 ≤6 位小数）
- **倍率范围**：1.01 - 10.00（输入 2 位小数）
- **庄家优势**：1%（RTP = 99%）
- **游戏模式**：手动、自动、极速
- **历史记录**：最近 25 局倍率结果
- **可证明公平**：完整的种子和验证机制

### 1.2 与 Dice 游戏的对比

| 特性 | Dice | Limbo |
|------|------|-------|
| 游戏类型 | 单局即时 | 单局即时 |
| 输入参数 | 目标数字 (4-96) + 方向 | 目标倍率 (1.01-10.00) |
| 结果生成 | 随机数 (0-100) | 随机倍率 (1.00-∞) |
| 判定逻辑 | roll >=/<= target | result >= target |
| 赔率设定 | 系统计算 | 玩家设定 |
| RTP | 97% | 99% |

## 2. 游戏规则

### 2.1 核心规则

**BR-001：游戏开始规则**
- 玩家输入有效下注金额和目标倍率
- 下注金额范围：0.000100 - 500,000 USD
- 目标倍率范围：1.01 - 10.00
- 系统校验余额充足性和赔付上限
- 同一玩家同一时刻仅允许一局进行

**BR-002：预测与判定规则**
- 胜利条件：结果倍率 ≥ 目标倍率
- 失败条件：结果倍率 < 目标倍率
- 胜利派彩：下注额 × 目标倍率
- 失败损失：全部下注额
- 结果由可证明公平系统生成

**BR-003：概率与期望值规则**
- 胜率计算：胜率 = (1 - 庄家优势) / 目标倍率 = 0.99 / 目标倍率
- 期望收益率：1 - 庄家优势 = 99%
- 目标倍率越高，胜率越低，风险越大

### 2.2 游戏模式

**手动模式**：
- 单次下注
- 显示火箭攀升动画（0.8-1.2秒）
- 支持快捷按钮（Min/÷2/×2/Max）

**极速模式**：
- 跳过动画，结果即时呈现（<150ms）
- 不影响游戏逻辑和结算
- 默认关闭，玩家可手动开启

**自动模式**（可选功能）：
- 轮数设置：1-10,000 轮
- 胜利后策略：重置/按百分比增加/按倍数增加
- 失败后策略：同胜利后策略
- 止盈止损：支持金额或百分比设置

## 3. 数学模型与概率

### 3.1 核心公式

```
胜率 = (1 - 庄家优势) / 目标倍率
胜率 = 0.99 / target_multiplier

期望收益率 = 1 - 庄家优势 = 99%

最高赔付 = 下注额 × 目标倍率（需 ≤ 系统上限）
```

### 3.2 随机倍率生成算法

**可证明公平机制**：
```
种子字符串 = clientSeed:serverSeed:nonce
哈希值 = SHA256(种子字符串)
哈希前8位 = hashHex[:8]
哈希十进制 = parseInt(哈希前8位, 16)
归一化值 = 哈希十进制 / 0xFFFFFFFF  // [0, 1)

// 倍率计算（指数分布）
result_multiplier = 0.99 / (1 - 归一化值)
result_multiplier = min(result_multiplier, 10)  // 上限保护
```

**概率验证**：
- 对于任意目标倍率 T
- P(result >= T) = P(0.99/(1-r) >= T) = P(r >= 1 - 0.99/T) = 0.99/T ✓

### 3.3 精度与取整

- **输入精度**：下注额 ≤6 位小数，倍率 2 位小数
- **内部计算**：使用 Decimal/BigInt 避免浮点误差
- **显示精度**：
  - 派彩默认 2 位小数，小额显示至 6 位
  - 历史倍率保留 2 位，支持千分位/科学计数法

## 4. 系统架构设计

### 4.1 游戏引擎（internal/biz/game/limbo/）

```go
type LimboGame struct {
    id               string
    state            engine.GameState
    aggregatorID     string
    userID           int64
    playerID         string
    sessionID        int64
    betAmount        float64
    currency         string
    targetMultiplier float64
    nonce            int64
    clientSeed       string
    serverSeed       string
    serverSeedID     int64
    hashedServerSeed string
}

type LimboOutcome struct {
    ResultMultiplier float64
    TargetMultiplier float64
}
```

**核心方法**：
- `Initialize(ctx)`: 初始化游戏状态
- `PlaceBet(ctx, playerID, amount, options)`: 验证参数并下注
- `Play(ctx)`: 执行游戏逻辑，生成结果
- `CalculateLimboMultiplier(clientSeed, serverSeed, nonce)`: 计算倍率
- `VerifyMultiplier(clientSeed, serverSeed, nonce)`: 验证结果

### 4.2 服务层（internal/service/games/limbo/）

```go
type Service struct {
    *base.SimpleGameService[UnifiedBetRequest, BetResponse]
    eventDispatcher *websocket.EventDispatcher
    betBroadcaster  *websocket.BetActivityBroadcaster
    logger          *log.Helper
}

type UnifiedBetRequest = games.UnifiedBetRequest[games.LimboParams]

type BetResponse struct {
    games.BaseBetResponse
    ResultMultiplier float64
    TargetMultiplier float64
}
```

**核心功能**：
- RPC 消息处理：通过 Centrifugo Proxy 处理游戏请求
- 游戏配置提供：`GetGameConfig`
- 活动广播：投注活动实时推送

### 4.3 API 定义（api/game/v1/）

**消息类型**：

```protobuf
// game_params.proto
message LimboParams {
  string target_multiplier = 1;  // 使用 string 存储高精度
}

// types.proto
message LimboOutcome {
  string result_multiplier = 1 [json_name = "resultMultiplier"];
  string target_multiplier = 2 [json_name = "targetMultiplier"];
}

// websocket.proto
message LimboGameConfig {
  int64 id = 1;
  string game_name = 2 [json_name = "gameName"];
  string game_id = 3 [json_name = "gameId"];
  string category = 4;
  string status = 5;
  string description = 6;
  string thumbnail = 7;
  string default_rtp = 8 [json_name = "defaultRTP"];
  repeated string features = 9;
  repeated LimboBetInfo bet_info = 10 [json_name = "betInfo"];
  LimboGameParameters game_parameters = 11 [json_name = "gameParameters"];
  string commission_rate = 12 [json_name = "commissionRate"];
  double max_reward_multiplier = 13 [json_name = "maxRewardMultiplier"];
}

message LimboBetInfo {
  string currency = 1;
  string currency_type = 2 [json_name = "currencyType"];
  double default_bet = 3 [json_name = "defaultBet"];
  double min_bet = 4 [json_name = "minBet"];
  double max_bet = 5 [json_name = "maxBet"];
  double max_profit = 6 [json_name = "maxProfit"];
}

message LimboGameParameters {
  string min_multiplier = 1 [json_name = "minMultiplier"];
  string max_multiplier = 2 [json_name = "maxMultiplier"];
  string default_multiplier = 3 [json_name = "defaultMultiplier"];
}
```

### 4.4 数据流

```
1. 客户端发送 PLACE_BET 消息
   ↓
2. WebSocket Handler 路由到 Limbo 服务
   ↓
3. 验证参数（倍率范围、下注金额等）
   ↓
4. 调用聚合器 API 扣款
   ↓
5. 获取/更新服务端种子和 nonce
   ↓
6. 游戏引擎生成随机倍率
   ↓
7. 判定胜负，计算派彩
   ↓
8. 保存游戏结果到数据库
   ↓
9. 如果获胜，调用聚合器 API 派彩
   ↓
10. 返回结果给客户端
   ↓
11. 广播投注活动和游戏结果
```

## 5. 配置管理

### 5.1 游戏配置（configs/games/limbo.json）

```json
{
  "gameId": "inhousegame:limbo",
  "status": "active",
  "gameParameters": {
    "multiplierRange": {
      "min": 1.01,
      "max": 10.0
    },
    "defaultMultiplier": 2.0
  },
  "rtp": 99.0
}
```

**配置说明**：
- `gameParameters.multiplierRange`：目标倍率的有效范围
- `gameParameters.defaultMultiplier`：无效输入时使用的默认倍率
- `rtp`：返还率（99.0 表示 99%，在代码中转换为 0.99）

### 5.2 常量定义（pkg/consts/games.go）

```go
const (
    GameIDLimbo = "inhousegame:limbo"
)
```

## 6. 可证明公平机制

### 6.1 种子管理

- **客户端种子**：玩家提供（8-256 字符）
- **服务端种子**：系统生成（加密存储）
- **哈希服务端种子**：SHA256(服务端种子)，公开展示
- **Nonce**：每次投注递增，确保唯一性

### 6.2 验证流程

1. 玩家可在游戏前查看哈希服务端种子
2. 游戏结束后，系统提供：
   - 客户端种子
   - 服务端种子（明文）
   - Nonce
   - 结果倍率
3. 玩家可使用公开算法验证：
   ```
   SHA256(clientSeed:serverSeed:nonce) → 结果倍率
   ```

## 7. 风控与限制

### 7.1 参数限制

- 目标倍率：1.01 - 10.00
- 下注金额：0.000100 - 500,000 USD
- 最高赔付：配置 maxPayoutAmount 上限
- 并发控制：同一玩家同时只能有一局进行

### 7.2 异常处理

- 余额不足：返回 INSUFFICIENT_BALANCE 错误
- 参数无效：返回 INVALID_PARAMETER 错误
- 超过赔付上限：返回 PAYOUT_LIMIT_EXCEEDED 错误
- 网络异常：支持幂等重试机制

## 8. 性能优化

- **缓存优化**：种子信息缓存
- **数据库优化**：nonce 原子更新（RETURNING 机制）
- **并发控制**：令牌机制防止重复下注
- **极速模式**：跳过动画，减少响应时间至 <150ms

## 9. 测试策略

### 9.1 单元测试

- 倍率生成算法正确性
- 概率分布验证（大样本测试）
- 可证明公平验证
- 边界条件测试（最小/最大倍率）

### 9.2 集成测试

- WebSocket 消息流程
- 聚合器 API 交互
- 数据库事务一致性
- 并发场景测试

### 9.3 性能测试

- 极速模式响应时间
- 高并发投注压力测试
- RTP 长期统计验证

## 10. 开发计划

### 阶段 1：文档和 API
- ✅ 详细设计文档
- ✅ WebSocket API 文档
- ✅ Protobuf 定义

### 阶段 2：核心实现
- ✅ 游戏引擎
- ✅ 服务层
- ✅ 配置和注册

### 阶段 3：测试和优化
- ✅ 单元测试
- ✅ 集成测试
- ✅ 性能优化

### 阶段 4：上线
- ✅ 文档更新
- ✅ 部署配置
- ✅ 监控告警

## 11. 附录

### 11.1 参考资料

- Limbo 游戏业务规则文档
- Dice 游戏实现代码
- 可证明公平标准规范

### 11.2 相关文档

- [Limbo WebSocket API 参考](./limbo-websocket-api-zh.md)
- [WebSocket API 索引](./websocket-api-reference-zh.md)
- [系统架构文档](./architecture-zh.md)