# HiLo（高低牌）游戏详细设计文档 ✅ 已实现

## 游戏规则

HiLo 是一个经典的卡牌预测游戏，玩家需要预测下一张牌比当前牌高还是低。

1. **牌值大小**：A(1) < 2 < 3 < 4 < 5 < 6 < 7 < 8 < 9 < 10 < J(11) < Q(12) < K(13)
2. **预测选项**：
   - Higher（更高）：预测下一张牌比当前牌大
   - Lower（更低）：预测下一张牌比当前牌小
   - Same（相同）：预测下一张牌与当前牌相同
   - Skip（跳过）：更换当前牌（仅在首次猜测前可用）
3. **特殊规则**：
   - 当前牌为 A 时，选择 "lower" 自动变为 "same"
   - 当前牌为 K 时，选择 "higher" 自动变为 "same"
4. **RTP**：99%

## 赔率计算公式

赔率基于概率动态计算，确保 99% 的 RTP：

```go
// 计算概率
func calculateProbability(currentCard int, choice string) float64 {
    switch choice {
    case "higher":
        return float64(13-currentCard) / 13.0
    case "lower":
        return float64(currentCard-1) / 13.0
    case "same":
        return 1.0 / 13.0
    }
}

// 计算乘数
func calculateMultiplier(probability float64) float64 {
    if probability == 0 {
        return 0
    }
    // RTP = 99%
    multiplier := 0.99 / probability
    return math.Round(multiplier*10000) / 10000  // 保留4位小数
}
```

**赔率示例**：
| 当前牌 | Higher概率 | Higher乘数 | Lower概率 | Lower乘数 |
|--------|-----------|-----------|----------|----------|
| A (1)  | 92.3%     | 1.0725x   | 0% (same) | 7.6154x  |
| 7      | 46.15%    | 2.1450x   | 46.15%   | 2.1450x  |
| K (13) | 0% (same) | 7.6154x   | 92.3%    | 1.0725x  |

## 游戏流程

1. **开始游戏**
   - 玩家设置投注金额和客户端种子
   - 系统发第一张牌
   - 游戏进入 playing 状态

2. **做出选择**
   - 玩家选择 higher/lower/skip/same
   - 系统生成下一张牌
   - 如果猜对，乘数累积，可以继续或兑现
   - 如果猜错，游戏结束，失去投注

3. **兑现**
   - 至少猜对一次后可兑现
   - 最终赔付 = 投注金额 × 累积乘数

4. **游戏恢复**
   - 支持断线重连
   - 保存完整游戏状态
   - 可恢复未完成的游戏

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/hilo/hilo.go
type GameService struct {
    serverSeedRepo  data.ServerSeedRepo
    gameSessionRepo data.GameSessionRepo
    gameResultRepo  data.GameResultRepo
    userRepo        data.UserRepo
    aggregatorRepo  data.AggregatorRepo
    clientManager   *aggregator.ClientManager
    idGenerator     *biz.IDGeneratorUsecase
    
    // 双索引缓存
    activeGamesByUser  map[int64]map[string]*GameInstance
    activeGamesByRound map[string]*GameInstance
}
```

### 2. 核心数据结构
```go
// internal/biz/game/hilo/hilo.go
type HiloGame struct {
    Status            GameStatus
    RoundID           string
    BetAmount         float64
    CurrentCard       int      // 当前牌 (1-13)
    CardHistory       []int    // 历史牌序列
    CurrentMultiplier float64  // 当前累积乘数
    MultiplierHistory []float64 // 乘数历史
    CanCashout        bool     // 是否可兑现
    GuessCount        int      // 猜测次数
    ClientSeed        string
    serverSeed        string   // 私密
    Nonce             int64
    FinalPayout       float64
    IsProfitable      bool
}

type HiloOutcome struct {
    CardHistory       []int     `json:"card_history"`
    MultiplierHistory []float64 `json:"multiplier_history"`
    FinalMultiplier   float64   `json:"final_multiplier"`
    GuessCount        int       `json:"guess_count"`
    FinalCard         int       `json:"final_card"`
    Status            string    `json:"status"`
}
```

### 3. WebSocket 接口集成
- **HILO_START_GAME**: 开始新游戏
- **HILO_MAKE_CHOICE**: 做出选择（higher/lower/skip/same）
- **HILO_CASH_OUT**: 兑现赢利
- **HILO_GET_STATE**: 获取游戏状态
- **HILO_CHECK_ACTIVE**: 检查活跃游戏
- **HILO_RESUME_GAME**: 恢复游戏

### 4. 会话管理特性
- **双索引缓存**：按用户ID和回合ID索引，提高查询效率
- **状态恢复**：支持从数据库恢复游戏状态
- **自动清理**：定期清理非活跃游戏，超时自动兑现
- **并发控制**：防止单用户多个并发游戏

### 5. 可证明公平实现
```go
func generateCardSequence(serverSeed, clientSeed string, nonce int64) []int {
    sequenceLength := 100  // 预生成100张牌
    cards := make([]int, sequenceLength)
    
    for i := 0; i < sequenceLength; i++ {
        // 创建哈希输入
        hashInput := fmt.Sprintf("%s:%s:%d:%d", serverSeed, clientSeed, nonce, i)
        hash := sha256.Sum256([]byte(hashInput))
        hashHex := hex.EncodeToString(hash[:])
        
        // 取前8字符转换为数字
        hashPart := hashHex[:8]
        num := new(big.Int)
        num.SetString(hashPart, 16)
        
        // 映射到牌值 (1-13)
        cardValue := num.Mod(num, big.NewInt(13)).Int64() + 1
        cards[i] = int(cardValue)
    }
    return cards
}
```

## 前端实现建议

1. **牌面显示**
   - 数字转换：1→A, 11→J, 12→Q, 13→K
   - 视觉效果：当前牌突出显示
   - 历史记录：显示所有已发的牌

2. **概率和乘数显示**
   - 实时显示 higher/lower 的概率
   - 动态更新可能获得的乘数
   - 特殊牌（A/K）的提示

3. **游戏控制**
   - Higher/Lower 按钮
   - Skip 按钮（首次猜测前）
   - Cash Out 按钮（至少猜对一次后）

4. **状态指示**
   - 当前累积乘数
   - 潜在赢利金额
   - 猜测次数统计