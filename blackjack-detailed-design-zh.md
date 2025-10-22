# Blackjack 游戏详细设计文档

## 1. 游戏概述

Blackjack（21点）是一款经典的纸牌游戏，玩家目标是让手牌点数尽可能接近21点但不超过，最终与庄家比较点数决定输赢。游戏支持要牌、停牌、加倍、分牌、保险等多种策略操作。

### 核心参数
- **目标点数**：21点
- **牌副数量**：6副标准扑克牌（不含大小王），共312张牌
- **庄家规则**：软17停牌（S17）
- **Blackjack赔率**：3:2
- **保险赔率**：2:1
- **理论RTP**：99.5%
- **游戏类型**：会话型游戏（session game）

## 2. 游戏规则

### 2.1 牌面点数计算
- **数字牌（2-9）**：按牌面数值计算点数
- **人物牌（J、Q、K）**：均为10点
- **A牌（Ace）**：可计算为1点或11点，取对玩家最有利的值
- **软牌型（Soft Hand）**：包含A且按11计算的牌型
- **硬牌型（Hard Hand）**：不包含A或A按1计算的牌型

### 2.2 游戏流程
1. **下注阶段**：玩家下注
2. **发牌阶段**：玩家和庄家各获得2张牌
   - 发牌顺序：玩家明牌 → 庄家明牌 → 玩家明牌 → 庄家暗牌
3. **玩家决策阶段**：玩家选择操作
4. **庄家操作阶段**：庄家按固定规则操作
5. **结算阶段**：比较点数，结算输赢

### 2.3 玩家操作

#### Hit（要牌）
- 从牌堆抽取1张牌加入手牌
- 可连续要牌直到满意或爆牌
- 超过21点即爆牌，立即败北

#### Stand（停牌）
- 保持当前手牌，结束玩家回合
- 轮到庄家操作

#### Double Down（加倍）
- **触发条件**：仅在前2张牌后可选择
- **操作**：投注金额翻倍，再抽1张牌后强制停牌
- **限制**：通常在9-11点时使用
- **分牌后**：可以对分出的每手牌分别加倍

#### Split（分牌）
- **触发条件**：前2张牌点数相同
- **操作**：拆分成2手牌，每手再下等额赌注
- **限制**：
  - A+A分牌后每手只能再要1张牌
  - 分牌后的A+10点牌算普通21点，非Blackjack
  - 最多只能分牌1次（不支持再次分牌）

#### Insurance（保险）
- **触发条件**：庄家明牌为A
- **保险金额**：原注金额的一半
- **结算**：庄家Blackjack时赔付2:1，否则保险金被收走

### 2.4 庄家操作规则
- **强制规则**：庄家必须按固定规则操作，无选择权
- **软17规则**：点数≤16必须要牌，≥17必须停牌
- **软17处理**：A+6=Soft 17必须停牌（S17规则）

### 2.5 结算规则

#### 结算优先级
1. **玩家爆牌**：立即败北，庄家收走赌注
2. **庄家爆牌**：所有未爆玩家获胜（1:1赔率）
3. **Blackjack对比**：
   - 玩家Blackjack vs 庄家非Blackjack：玩家胜（3:2赔率）
   - 玩家非Blackjack vs 庄家Blackjack：庄家胜
   - 双方Blackjack：平局（Push）
4. **点数对比**：双方都≤21时比较点数高低
   - 玩家点数高：玩家胜（1:1赔率）
   - 庄家点数高：庄家胜
   - 点数相等：平局（Push），退还本金

#### 赔率表
| 情况 | 赔率 |
|------|------|
| 普通获胜 | 1:1 |
| Blackjack获胜 | 3:2 |
| 保险成功 | 2:1 |
| 分牌后21点 | 1:1 |
| 加倍获胜 | 双倍投注按对应赔率 |

## 3. 技术实现

### 3.1 可证明公平性

#### 牌序生成算法
```go
// 使用服务器种子、客户端种子和nonce生成6副牌序列
func generateDeckSequence(serverSeed, clientSeed string, nonce int64) []Card {
    deckCount := 6
    totalCards := deckCount * 52
    sequence := make([]Card, totalCards)
    
    // 初始化6副牌
    deck := initializeDecks(deckCount)
    
    // Fisher-Yates洗牌算法
    for i := totalCards - 1; i > 0; i-- {
        hashInput := fmt.Sprintf("%s:%s:%d:%d", serverSeed, clientSeed, nonce, i)
        hash := sha256.Sum256([]byte(hashInput))
        hashHex := hex.EncodeToString(hash[:])
        
        // 使用hash值生成随机索引
        j := deriveIndex(hashHex, i+1)
        deck[i], deck[j] = deck[j], deck[i]
    }
    
    return deck
}
```

#### 验证机制
- 每局游戏开始时生成完整的312张牌序列
- 发牌顺序严格按照生成的序列
- 游戏结束后可通过种子信息验证牌序

### 3.2 状态管理

#### 游戏状态
```go
type GameStatus string

const (
    StatusBetting    GameStatus = "betting"     // 下注中
    StatusDealing    GameStatus = "dealing"     // 发牌中
    StatusPlaying    GameStatus = "playing"     // 玩家操作中
    StatusDealer     GameStatus = "dealer"      // 庄家操作中
    StatusFinished   GameStatus = "finished"    // 游戏结束
    StatusCashedOut  GameStatus = "cashed_out"  // 已结算
)
```

#### 手牌状态
```go
type HandStatus string

const (
    HandActive    HandStatus = "active"     // 可操作
    HandStand     HandStatus = "stand"      // 已停牌
    HandBusted    HandStatus = "busted"     // 爆牌
    HandBlackjack HandStatus = "blackjack"  // 天生21点
    HandWin       HandStatus = "win"        // 获胜
    HandLose      HandStatus = "lose"       // 失败
    HandPush      HandStatus = "push"       // 平局
)
```

### 3.3 数据结构

#### 核心游戏结构
```go
type Game struct {
    game.Game    // 嵌入基类（包含 RoundID, BetAmount, ClientSeed, ServerSeed, Nonce, FinalPayout, RTP）

    // 游戏状态
    Status          GameStatus
    PlayerHands     []PlayerHand
    ActiveHandIndex int
    DealerHand      DealerHand
    Insurance       *Insurance
    CanInsure       bool

    // 牌序
    DeckSequence    []Card
    NextCardIdx     int

    // 结算信息
    TotalBet        float64
    TotalPayout     float64
}
```

**架构变更**：
- 重命名 `BlackjackGame` 为 `Game`
- 嵌入 `game.Game` 基类,复用通用字段
- 移除冗余字段(`IsProfitable`等),统一使用基类的 `FinalPayout`

#### 玩家手牌结构
```go
type PlayerHand struct {
    Cards         []Card
    Status        HandStatus
    BetAmount     float64

    // 状态标记
    IsDoubled     bool  // 是否已加倍
    IsSplit       bool  // 是否由分牌产生
    IsFromAces    bool  // 是否由A+A分牌产生

    // 计算值
    SoftValue     int   // 软牌点数
    HardValue     int   // 硬牌点数
    BestValue     int   // 最佳点数（不超过21的最大值）
    IsBlackjack   bool  // 是否天生21点

    // 可用操作
    CanHit        bool
    CanStand      bool
    CanDouble     bool
    CanSplit      bool

    // 结算信息（仅游戏结束时返回）
    Payout        float64  // 赔付金额
    Result        string   // 结果：win/lose/push
}
```

**注意**：`Payout` 和 `Result` 字段只在游戏结束时返回给客户端。

#### 庄家手牌结构
```go
type DealerHand struct {
    Cards         []Card
    Status        HandStatus
    ShowCard      *Card   // 明牌（指针类型）
    HoleCard      *Card   // 暗牌（指针类型）

    // 计算值
    SoftValue     int
    HardValue     int
    BestValue     int

    // 状态
    IsRevealed    bool    // 暗牌是否已翻开
    HasBlackjack  bool
}
```

### 3.4 操作流程

#### 游戏开始流程
1. 验证客户端种子（8-256字符）
2. 获取或创建服务器种子
3. 生成6副牌序列
4. 扣除投注金额
5. 发4张初始牌
6. 检查Blackjack
7. 如庄家明牌为A，提供保险选项

#### 玩家操作流程
1. 验证当前手牌是否可操作
2. 执行相应操作（Hit/Stand/Double/Split）
3. 更新手牌状态
4. 检查是否爆牌
5. 如有多手牌，切换到下一手
6. 所有手牌完成后，进入庄家阶段

#### 庄家操作流程
1. 翻开暗牌
2. 按S17规则自动操作
3. 点数≤16时继续要牌
4. 点数≥17时停牌
5. 检查是否爆牌

#### 结算流程
1. 比较每手牌与庄家点数
2. 计算各手牌输赢
3. 处理保险结算
4. 计算总赔付
5. 更新玩家余额
6. 保存游戏结果

## 4. WebSocket消息设计

### 4.1 消息类型

#### 请求消息
- `PLACE_BET` - 开始游戏（gameParams.blackjack 为空对象）
- `BLACKJACK_HIT` - 要牌
- `BLACKJACK_STAND` - 停牌
- `BLACKJACK_DOUBLE` - 加倍
- `BLACKJACK_SPLIT` - 分牌
- `BLACKJACK_INSURANCE` - 购买保险
- `BLACKJACK_GET_STATE` - 获取游戏状态
- `BLACKJACK_CHECK_ACTIVE` - 检查活跃游戏
- `BLACKJACK_RESUME_GAME` - 恢复游戏

#### 响应消息
- `BLACKJACK_GAME_STATE` - 统一的游戏状态响应
- `BLACKJACK_CHECK_ACTIVE_RESPONSE` - 检查活跃游戏响应
- `BLACKJACK_RESUME_GAME_RESPONSE` - 恢复游戏响应

**注意**：不再使用单独的 `BLACKJACK_DEALER_REVEAL` 和 `BLACKJACK_GAME_RESULT` 消息类型,所有状态更新统一使用 `BLACKJACK_GAME_STATE`。

### 4.2 状态推送策略

#### 游戏进行中
- 推送 `BLACKJACK_GAME_STATE`,包含游戏状态和可用操作
- `playerHands` 不包含 `payout` 和 `result` 字段
- 不暴露输赢信息,遵循会话游戏隐私保护设计

#### 游戏结束后
- 推送 `BLACKJACK_GAME_STATE`,包含完整结算信息
- `playerHands` 包含 `payout` 和 `result` 字段
- 返回 `totalPayout` 和 `finalPayout` 字段
- 移除了冗余的 `netProfit` 字段（客户端可自行计算）

## 5. 特殊规则处理

### 5.1 分牌规则
- 仅支持相同点数的牌分牌
- A+A分牌后每手只能要1张牌
- 分牌后的21点不算Blackjack
- 最多分牌1次（2手牌）

### 5.2 加倍规则
- 仅在前2张牌时可用
- 通常限制在9-11点
- 分牌后的手牌可单独加倍
- 加倍后只能要1张牌

### 5.3 保险规则
- 仅在庄家明牌为A时提供
- 保险金额为原注的50%
- 独立于主游戏结算
- 最大赔付2:1

## 6. 性能优化

### 6.1 缓存策略
- 活跃游戏缓存在内存
- 超时自动清理（30分钟）
- 支持断线重连恢复

### 6.2 并发处理
- 使用读写锁保护游戏状态
- 批量操作使用事务
- 异步处理非关键操作

## 7. 测试要点

### 7.1 规则测试
- 所有操作的合法性验证
- 点数计算正确性
- 结算逻辑准确性
- 特殊规则处理

### 7.2 边界测试
- 爆牌处理
- Blackjack识别
- 分牌限制
- 保险结算

### 7.3 RTP验证
- 大量模拟验证RTP
- 各种策略下的收益率
- 确保达到99.5%理论值

## 8. 配置参数

```json
{
  "gameId": "inhousegame:blackjack",
  "gameName": "Blackjack",
  "category": "session",
  "defaultRTP": "99.5%",
  "features": ["provably_fair", "session_game", "multi_hand", "insurance"],
  "gameParameters": {
    "deckCount": 6,
    "dealerRule": "S17",
    "blackjackPayout": 1.5,
    "insurancePayout": 2,
    "maxSplits": 1,
    "doubleAfterSplit": true,
    "surrenderAllowed": false
  }
}
```

## 9. 实现状态

- ✅ 游戏引擎核心逻辑
- ✅ WebSocket消息处理
- ✅ 多手牌管理
- ✅ 保险机制
- ✅ 可证明公平性验证
- ✅ 断线重连支持