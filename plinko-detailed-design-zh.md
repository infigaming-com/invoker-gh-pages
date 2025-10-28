# Plinko（弹珠台）游戏详细设计文档 ✅ 已实现

## 游戏概述

Plinko 是一个经典的弹珠掉落游戏，小球从顶部释放，通过钉桩阵列向下弹跳，最终落入底部不同槽位获得相应倍率奖励。

**特殊机制**：Plinko 采用延迟结算机制，确保在动画播放期间 Invoker 前端和聚合器前端的余额始终保持一致，提升用户体验。详见 [延迟结算机制](#延迟结算机制) 章节。

## 游戏规则

### 基本玩法
1. **参数选择**：
   - 行数：8-16行（可选：8、10、12、14、16）
   - 难度：low（低风险）、medium（中风险）、high（高风险）
   - 默认配置：12行，medium难度

2. **游戏流程**：
   - 设置投注金额和游戏参数
   - 小球从顶部中心释放
   - 经过每行钉桩随机向左或右弹跳
   - 最终落入底部槽位
   - 根据槽位获得对应倍率奖励

3. **概率分布**：
   - 中心槽位概率最高，倍率较低
   - 边缘槽位概率最低，倍率较高
   - 符合二项式分布规律

## 赔率配置

### Low难度倍率表

| 行数 | 边缘倍率 | 中心倍率 | 倍率范围 |
|------|----------|----------|----------|
| 8    | 5.6×     | 0.5×     | 0.5-5.6× |
| 10   | 8.9×     | 0.5×     | 0.5-8.9× |
| 12   | 10×      | 0.9×     | 0.9-10×  |
| 14   | 7.1×     | 0.9×     | 0.9-7.1× |
| 16   | 16×      | 0.9×     | 0.9-16×  |

### Medium难度倍率表

| 行数 | 边缘倍率 | 中心倍率 | 倍率范围 |
|------|----------|----------|----------|
| 8    | 12.9×    | 0.4×     | 0.4-12.9× |
| 10   | 22×      | 0.5×     | 0.5-22×   |
| 12   | 16.5×    | 0.4×     | 0.4-16.5× |
| 14   | 58×      | 0.5×     | 0.5-58×   |
| 16   | 110×     | 0.3×     | 0.3-110×  |

### High难度倍率表

| 行数 | 边缘倍率 | 中心倍率 | 倍率范围 |
|------|----------|----------|-----------|
| 8    | 29×      | 0.2×     | 0.2-29×   |
| 10   | 76×      | 0.2×     | 0.2-76×   |
| 12   | 141×     | 0.2×     | 0.2-141×  |
| 14   | 56×      | 0.1×     | 0.1-56×   |
| 16   | 1000×    | 0.2×     | 0.2-1000× |

## 小球路径算法

### 路径生成
```go
func CalculatePath(clientSeed, serverSeed string, nonce int64, rows int) []int {
    // 生成种子哈希
    seedStr := fmt.Sprintf("%s:%s:%d", clientSeed, serverSeed, nonce)
    hash := sha256.Sum256([]byte(seedStr))
    
    path := make([]int, rows)
    for i := 0; i < rows; i++ {
        // 每行使用1位决定方向
        byteIndex := i / 8
        bitIndex := i % 8
        
        // 扩展哈希以支持超过256位
        if byteIndex >= len(hash) {
            extendedSeed := fmt.Sprintf("%s:%d", seedStr, byteIndex/32)
            hash = sha256.Sum256([]byte(extendedSeed))
        }
        
        // 0=左，1=右
        bit := (hash[byteIndex] >> bitIndex) & 1
        path[i] = int(bit)
    }
    
    return path
}
```

### 最终位置计算
- 起始位置：顶部中心（位置0）
- 每次向右移动：位置+1
- 每次向左移动：位置不变
- 最终槽位 = 所有移动之和

## 概率计算

### 二项式分布
```
P(k) = C(n,k) / 2^n
```
其中：
- n = 行数
- k = 最终槽位位置
- C(n,k) = 组合数

### 概率示例（12行）

| 槽位 | 概率 | 期望频率 |
|------|------|----------|
| 0/12 | 0.024% | 1/4096 |
| 1/11 | 0.29% | 12/4096 |
| 2/10 | 1.61% | 66/4096 |
| 3/9  | 5.37% | 220/4096 |
| 4/8  | 12.08% | 495/4096 |
| 5/7  | 19.34% | 792/4096 |
| 6    | 22.56% | 924/4096 |

## 可证明公平实现

### 核心组件
- **客户端种子**：玩家提供（8-256字符）
- **服务端种子**：系统生成，游戏前提供哈希
- **Nonce**：回合计数器
- **验证函数**：允许事后验证结果

### 验证流程
```go
func VerifyPlinkoFairness(
    clientSeed string,
    serverSeed string,
    serverSeedHash string,
    nonce int64,
    rows int,
    difficulty string,
) (*VerificationResult, error) {
    // 1. 验证服务端种子哈希
    if !verifyServerSeedHash(serverSeed, serverSeedHash) {
        return nil, ErrInvalidServerSeed
    }
    
    // 2. 重新计算路径
    path := CalculatePath(clientSeed, serverSeed, nonce, rows)
    
    // 3. 计算最终位置和倍率
    finalSlot := sum(path)
    multiplier := getMultiplier(rows, difficulty, finalSlot)
    
    // 4. 计算理论概率
    probability := calculateProbability(rows, finalSlot)
    
    return &VerificationResult{
        Path:        path,
        FinalSlot:   finalSlot,
        Multiplier:  multiplier,
        Probability: probability,
    }, nil
}
```

## 延迟结算机制

### 设计背景

Plinko 游戏具有较长的动画播放时间（小球下落路径动画），如果在下注时立即结算，会导致以下问题：

1. **余额不一致问题**：
   - Invoker 前端余额已更新，但聚合器前端余额未同步
   - 动画播放期间两个前端显示不一致，用户体验差

2. **前端崩溃风险**：
   - 如果前端在动画期间崩溃，可能导致结算丢失
   - 需要可靠的兜底机制确保结算完成

### 三阶段流程

#### 阶段 1：下注（PLACE_BET）

**流程**：
1. **调用聚合器**：发送 Bet 扣款请求，扣除下注金额
2. **计算结果**：在本地计算游戏结果（路径、倍率、赔付）
3. **保存状态**：在数据库中保存游戏结果，状态为 `pending`
4. **返回响应**：返回完整游戏结果（路径、倍率），但 **不返回最终余额**
5. **前端行为**：前端 **不更新余额显示**，保持与聚合器前端一致

**关键代码**（`internal/service/games/instant_game.go:289-316`）：
```go
if needsDelayedSettlement {
    // 延迟结算：先扣款（Bet），动画完成后再派奖（Win + End）
    actions := []aggregator.PlayAction{
        {
            Action:        aggregator.PlayActionBet,
            TxnID:         betTxnID,
            UpdateBalance: true,
            Finished:      false,
            Amount:        amount.StringFixed(8),
            BetTime:       gameResult.CreatedAt,
        },
    }
    currentBalance, err = client.Play(ctx, authInfo, actions)
    // 延迟结算游戏不更新 balanceSyncer，前端不显示扣款后的余额
}
```

**聚合器请求示例**：
```json
{
  "actions": [
    {
      "action": "Bet",
      "txnId": "1234567890",
      "amount": "10.00",
      "updateBalance": true,
      "finished": false
    }
  ]
}
```

#### 阶段 2：动画播放

**特点**：
- 前端播放小球下落动画（约 2-5 秒）
- 此时 Invoker 前端和聚合器前端的余额都显示扣款后的金额（例如 1000.00）
- **两个前端保持一致**，用户体验流畅
- 服务端无操作，游戏状态保持 `pending`

#### 阶段 3：结算（SETTLE_BET）

**流程**：
1. **触发时机**：动画播放完成后，前端发送 `SETTLE_BET` 请求
2. **调用聚合器**：发送 Win（如果赢）+ End 请求，完成赔付
3. **更新状态**：将游戏结果状态更新为 `completed`
4. **并行推送**：聚合器通过 WebSocket 同时推送余额更新到两个前端
5. **最终一致**：两个前端同时更新余额显示（例如 1018.00）

**关键代码**（`internal/service/games/instant_game.go:424-588`）：
```go
func (s *InstantGameService) SettleBet(ctx context.Context, conn *wsTransport.UserConnection, req *gamev1.SettleBetRequest) error {
    // 1. 查询游戏结果
    gameResult, err := s.GameResultRepo.GetByID(ctx, req.RoundId)

    // 2. 幂等性检查
    if gameResult.Status == consts.GameResultStatusCompleted {
        // 已结算，直接返回余额
        currentBalance, _ := client.GetBalance(ctx, authInfo)
        return conn.SendMessage(wsResponse)
    }

    // 3. 调用聚合器结算
    actions := []aggregator.PlayAction{}
    if gameResult.FinalPayout.GreaterThan(decimal.Zero) {
        actions = append(actions, aggregator.PlayAction{
            Action:        aggregator.PlayActionWin,
            TxnID:         winTxnID,
            Amount:        gameResult.FinalPayout.StringFixed(8),
            UpdateBalance: true,
        })
    }
    actions = append(actions, aggregator.PlayAction{
        Action:   aggregator.PlayActionEnd,
        TxnID:    endTxnID,
        Finished: true,
    })

    // 4. 更新游戏状态
    gameResult.Status = consts.GameResultStatusCompleted
    s.GameResultRepo.Update(ctx, gameResult)

    // 5. 广播投注活动
    if gameResult.BetAmount.GreaterThan(decimal.Zero) {
        s.BetBroadcaster.PublishBetActivity(&websocket.BetActivity{...})
    }

    return nil
}
```

**聚合器请求示例**：
```json
{
  "actions": [
    {
      "action": "Win",
      "txnId": "1234567891",
      "amount": "18.00",
      "updateBalance": true
    },
    {
      "action": "End",
      "txnId": "1234567892",
      "finished": true
    }
  ]
}
```

### 异常兜底机制

#### 定时任务（PendingGameSettler）

**职责**：扫描超时的 pending 游戏并自动结算

**核心参数**：
```go
const (
    settlementTimeoutSeconds = 20   // 超时阈值：20秒
    batchSize                = 100  // 批量处理数量
    tickerInterval           = 20 * time.Second  // 扫描间隔
)
```

**工作流程**：
1. **定时扫描**：每 20 秒执行一次
2. **查询条件**：`status = 'pending' AND created_at < now() - 20s`
3. **自动结算**：调用聚合器 Play API 完成结算（Win + End）
4. **更新状态**：标记为 `completed`
5. **广播活动**：发布投注活动事件

**实现文件**：`internal/service/scheduler/pending_game_settler.go`

**启动方式**（`cmd/server/main.go:35-39`）：
```go
func newApp(..., settler *scheduler.PendingGameSettler) *kratos.App {
    // 启动定时任务
    if err := settler.Start(context.Background()); err != nil {
        log.NewHelper(logger).Fatalf("Failed to start pending game settler: %v", err)
    }
    // ...
}
```

#### 幂等性保护

**场景**：客户端重复发送 SETTLE_BET 请求

**处理逻辑**：
```go
// 如果游戏已经结算，查询聚合器获取最新余额
if gameResult.Status == consts.GameResultStatusCompleted {
    currentBalance, err := client.GetBalance(ctx, authInfo)
    // 返回最新余额
    return conn.SendMessage(wsResponse)
}
```

**保障**：
- 不会重复调用聚合器结算
- 不会重复扣款或派奖
- 返回最新余额供前端显示

### 状态管理

#### 数据库字段

**GameResult 模型**（`internal/data/models/game_result.go`）：
```go
type GameResult struct {
    // ... 其他字段
    Status string `gorm:"column:status;type:varchar(20);not null;default:'pending';index;comment:状态(pending/completed)"`
}
```

#### 状态常量

**定义位置**（`pkg/consts/games.go`）：
```go
const (
    GameResultStatusPending   = "pending"   // 待结算
    GameResultStatusCompleted = "completed" // 已完成
)
```

#### 状态流转

```
pending（PLACE_BET 时）→ completed（SETTLE_BET 或定时任务后）
```

### Handler 接口

**接口定义**（`internal/service/games/instant_game.go`）：
```go
type IGameHandler interface {
    // ... 其他方法
    NeedsDelayedSettlement() bool  // 返回 true 表示需要延迟结算
}
```

**Plinko 实现**（`internal/service/games/plinko/handler.go`）：
```go
func (h *Handler) NeedsDelayedSettlement() bool {
    return true  // Plinko 需要延迟结算
}
```

**其他游戏**：
- Dice: `return false`
- Keno: `return false`
- Limbo: `return false`
- DragonTiger: `return false`
- Roulette: `return false`

### API 差异

| 字段 | PLACE_BET 响应 | SETTLE_BET 响应 |
|------|----------------|-----------------|
| `roundId` | ✅ 返回 | ✅ 返回 |
| `gameResult` | ✅ 返回完整结果（路径、倍率等） | ❌ 不返回 |
| `balance` | ❌ 不返回 | ✅ 返回最终余额 |
| `hasAnimation` | ✅ true | ❌ 不返回 |

### 投注活动广播优化

**设计原则**：
- **即时游戏**：在 PLACE_BET 响应后立即广播
- **延迟结算游戏**：在 SETTLE_BET 完成后才广播
- **过滤条件**：金额为 0 的测试投注不进行广播

**代码位置**（`instant_game.go:401-419`）：
```go
// 即时游戏：立即广播
if !needsDelayedSettlement && amount.GreaterThan(decimal.Zero) {
    s.BetBroadcaster.PublishBetActivity(&websocket.BetActivity{
        PlayerID:     userInfo.PlayerID,
        GameID:       user.GameID,
        BetAmount:    amount,
        Payout:       outcome.GetFinalPayout(),
        Multiplier:   outcome.GetMultiplier(),
        // ...
    })
}
```

**代码位置**（`instant_game.go:549-569`）：
```go
// 延迟结算游戏：在 SETTLE_BET 后广播
if gameResult.BetAmount.GreaterThan(decimal.Zero) {
    s.BetBroadcaster.PublishBetActivity(&websocket.BetActivity{
        PlayerID:   gameResult.PlayerID,
        GameID:     gameResult.GameID,
        BetAmount:  gameResult.BetAmount,
        Payout:     gameResult.FinalPayout,
        Multiplier: gameResult.Multiplier,
        // ...
    })
}
```

### 设计优势

1. **用户体验优秀**：
   - 动画期间两个前端余额完全一致
   - 不会看到余额跳变，体验流畅

2. **可靠性高**：
   - 定时任务兜底，防止长时间 pending
   - 幂等保护，支持重复结算请求

3. **性能优化**：
   - 异步结算不阻塞动画播放
   - 批量处理超时游戏，减少数据库压力

4. **灵活扩展**：
   - 其他需要动画的游戏可以轻松启用
   - 只需实现 `NeedsDelayedSettlement()` 返回 true

### 相关文档

- [Plinko WebSocket API](./plinko-websocket-api-zh.md) - SETTLE_BET 接口详细说明
- [序列图](./sequence-diagrams-zh.md) - 延迟结算流程图

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/plinko/handler.go
type GameService struct {
    serverSeedRepo  data.ServerSeedRepo
    gameResultRepo  data.GameResultRepo
    userRepo        data.UserRepo
    aggregatorRepo  data.AggregatorRepo
    clientManager   *aggregator.ClientManager
    idGenerator     *biz.IDGeneratorUsecase
}
```

### 2. 核心数据结构
```go
// 游戏实例
type PlinkoGame struct {
    id               string
    state            GameState
    engine           RandomProvider
    rows             int        // 行数
    difficulty       string     // 难度
    betAmount        float64    // 投注金额
    nonce            int64      // Nonce值
    clientSeed       string     // 客户端种子
    serverSeed       string     // 服务端种子
    
    // 结果
    path             []int      // 路径序列
    finalSlot        int        // 最终槽位
    multiplier       float64    // 获得倍率
    payout           float64    // 奖金
}

// 游戏结果
type PlinkoOutcome struct {
    Rows       int      `json:"rows"`       // 行数
    Difficulty string   `json:"difficulty"` // 难度
    Path       []int    `json:"path"`       // 小球路径
    FinalSlot  int      `json:"finalSlot"`  // 最终槽位
    Multiplier float64  `json:"multiplier"` // 获得倍率
}
```

### 3. WebSocket接口
- **PLACE_BET**：下注并执行游戏
  - 参数：`rows`、`difficulty`、`betAmount`、`clientSeed`
  - 返回：完整游戏结果

### 4. 倍率配置管理

倍率配置通过结构化的 Protobuf 消息传递给客户端：

```protobuf
message PlinkoGameParameters {
  map<string, PlinkoPayoutTable> payout_tables = 7;  // 按难度分类
}

message PlinkoPayoutTable {
  map<string, PlinkoRowMultipliers> rows = 1;  // 按行数分类
}

message PlinkoRowMultipliers {
  repeated double multipliers = 1;  // 倍率数组
}
```

**示例：通过 GET_GAME_CONFIG 获取配置**
```json
{
  "gameParameters": {
    "payoutTables": {
      "low": {
        "rows": {
          "8": { "multipliers": [5.6, 2.1, 1.1, 1, 0.5, 1, 1.1, 2.1, 5.6] },
          "10": { "multipliers": [8.9, 3, 1.4, 1.1, 1, 0.5, 1, 1.1, 1.4, 3, 8.9] }
        }
      },
      "medium": { /* ... */ },
      "high": { /* ... */ }
    }
  }
}
```

**内部实现**：
```go
// internal/biz/game/plinko/plinko.go
var multiplierConfig = map[string]map[int][]float64{
    "low": {
        8:  {5.6, 2.1, 1.1, 1, 0.5, 1, 1.1, 2.1, 5.6},
        10: {8.9, 3, 1.4, 1.1, 1, 0.5, 1, 1.1, 1.4, 3, 8.9},
        // ... 更多配置
    },
    "medium": { /* ... */ },
    "high": { /* ... */ },
}
```

## RTP（理论返还率）

### RTP计算公式
```go
func CalculateRTP(difficulty string, rows int) float64 {
    multipliers := getMultipliers(difficulty, rows)
    slots := len(multipliers)
    totalRTP := 0.0
    
    for slot, multiplier := range multipliers {
        probability := calculateProbability(rows, slot)
        totalRTP += probability * multiplier
    }
    
    return totalRTP
}
```

### 理论RTP
- 设计目标：约99%
- 实际值：98.5%-99.5%（根据配置微调）
- House Edge：约1%

## 前端实现建议

### 1. 界面设计
- **游戏板**：
  - 三角形钉桩阵列
  - 底部槽位显示倍率
  - 高亮当前选中配置

- **控制面板**：
  - 行数选择器（滑块）
  - 难度选择（按钮组）
  - 投注金额输入
  - 客户端种子输入

### 2. 动画效果
- **小球掉落**：
  - 物理引擎模拟真实弹跳
  - 路径高亮显示
  - 速度可调节

- **获胜动画**：
  - 槽位发光效果
  - 倍率放大显示
  - 金币飞溅特效

### 3. 信息显示
- **实时信息**：
  - 当前路径
  - 预测槽位
  - 潜在奖金

- **历史记录**：
  - 最近10次结果
  - 路径可视化
  - 倍率统计

### 4. 公平性验证
- 显示服务端种子哈希
- 提供验证工具链接
- 导出游戏数据功能

## 性能优化

1. **配置预加载**：
   - 启动时加载所有倍率配置
   - 缓存常用计算结果

2. **批量处理**：
   - 支持连续自动投注
   - 批量结果计算

3. **内存优化**：
   - 使用对象池复用游戏实例
   - 及时清理历史数据