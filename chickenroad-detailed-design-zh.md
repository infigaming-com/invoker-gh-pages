# ChickenRoad（小鸡过马路）游戏详细设计文档 ✅ 已实现

## 配置文件

### 游戏配置 (`configs/games/chickenroad.json`)
```json
{
  "gameId": "inhousegame:chickenroad",
  "status": "active",
  "oddsType": "table",
  "rtp": 98.0,
  "gameParameters": {
    "defaultDifficulty": "easy"
  },
  "payoutTables": {
    "easy": {
      "multipliers": [
        {"step": 1, "multiplier": 1.03}, {"step": 2, "multiplier": 1.07}, {"step": 3, "multiplier": 1.12},
        ...
        {"step": 24, "multiplier": 19.44}
      ]
    },
    "medium": { ... },
    "hard": { ... },
    "daredevil": { ... }
  }
}
```

### 配置说明
- **gameId**: 游戏唯一标识符，格式为 `"inhousegame:chickenroad"`
- **status**: 游戏状态（`active` 启用，`disabled` 禁用）
- **oddsType**: 赔率类型，`table` 表示表驱动（赔率由 payoutTables 决定）
- **rtp**: 理论回报率（98%），用于计算存活概率（存活概率 = RTP / 倍率）
- **gameParameters**: 游戏参数
  - **defaultDifficulty**: 默认难度（`easy`）
- **payoutTables**: 赔率表配置，支持 4 种难度模式
  - `easy`: 简单模式（24步，低风险，最高倍率 19.44×）
  - `medium`: 中等模式（22步，中等风险，最高倍率 1788.80×）
  - `hard`: 困难模式（20步，高风险，最高倍率 41321.43×）
  - `daredevil`: 魔鬼模式（15步，极高风险，最高倍率 2542251.93×）
  - 赔率表格式：`{ "multipliers": [{"step": N, "multiplier": M}, ...] }`

### 配置加载
- **加载时机**: 服务器启动时，GameRegistry 自动从 `configs/games/` 目录加载所有 `.json` 配置文件
- **使用方式**: 游戏服务通过 `gameRegistry.GetGame("inhousegame:chickenroad")` 获取配置，加载赔率表用于游戏结算
- **配置验证**: Service 层和 Biz 层都会验证配置完整性，配置缺失时返回错误
- **投注限额**: 从 `configs/currencies.json` 自动生成 BetInfo

### 难度特征对比

| 难度 | 最大步数 | 最低倍率 | 最高倍率 | 风险等级 |
|------|----------|----------|----------|----------|
| Easy | 24 | 1.03× | 19.44× | 低 |
| Medium | 22 | 1.12× | 1788.80× | 中 |
| Hard | 20 | 1.23× | 41321.43× | 高 |
| Daredevil | 15 | 1.63× | 2542251.93× | 极高 |

## 游戏概述

ChickenRoad 是一个策略性会话游戏，玩家引导小鸡逐步穿越危险的马路，每成功前进一步都会增加赔率倍数，但同时也面临失败的风险。

## 游戏规则

### 基本玩法
1. **开始游戏**：
   - 选择难度级别
   - 设置投注金额
   - 输入客户端种子

2. **游戏进行**：
   - 逐步选择前进
   - 每步成功获得倍数增长
   - 遇到危险游戏结束
   - 可随时兑现当前倍数

3. **游戏结束**：
   - 成功兑现：获得累积倍数奖励
   - 遇到危险：失去全部投注
   - 完成所有步数：获得最大倍数

### 游戏状态
- `ready` - 准备就绪
- `playing` - 游戏进行中
- `finished` - 游戏失败结束
- `cashed_out` - 成功兑现

## 难度配置

### 难度级别对比

| 难度 | 最大步数 | RTP | 风险等级 |
|------|----------|-----|----------|
| **Easy** | 19 | 98% | 低 |
| **Medium** | 17 | 98% | 中 |
| **Hard** | 15 | 98% | 高 |
| **Expert** | 10 | 98% | 极高 |

### 倍率与获胜概率计算公式

**统一 RTP**: 98%
**初始获胜率**: 1.0

**计算公式**:
```
获胜率 = (总步数 - 当前步数 + 1) / (20 - 当前步数 + 1) × 上一个获胜率
倍率 = (1 ÷ 获胜率) × 0.98
```

**精度要求**:
- 倍率保留小数点后2位，四舍五入
- 获胜率保留小数点后6位，四舍五入

### 倍率计算示例

倍率和概率通过上述公式动态计算，每个难度级别根据最大步数自动生成倍率序列。

**计算示例（Easy难度，19步）**:
- 第1步: 获胜率 = (19-1+1)/(20-1+1) × 1.0 = 0.95, 倍率 = (1/0.95) × 0.98 = 1.03×
- 第10步: 获胜率根据递推计算，倍率相应调整
- 第19步: 达到最大步数，获得最终倍率

**不同难度的风险特征**:
- **Easy**: 步数多，单步倍率增长缓慢，适合稳健玩家
- **Medium**: 中等步数，风险与收益平衡
- **Hard**: 步数较少，倍率增长较快，高风险高回报
- **Expert**: 仅10步，倍率增长极快，极高风险

## 路径生成算法

### 预生成机制
```go
func generateStepResults(clientSeed, serverSeed string, nonce int64, maxSteps int) []bool {
    results := make([]bool, maxSteps)
    
    for i := 0; i < maxSteps; i++ {
        // 生成步骤哈希
        seedStr := fmt.Sprintf("%s:%s:%d:%d", clientSeed, serverSeed, nonce, i)
        hash := sha256.Sum256([]byte(seedStr))
        hashHex := hex.EncodeToString(hash[:])
        
        // 转换为数值
        hashValue := new(big.Int)
        hashValue.SetString(hashHex[:16], 16)
        
        // 计算当前步成功概率
        probability := calculateStepProbability(i, difficulty)
        threshold := new(big.Int).Mul(
            big.NewInt(int64(probability * 1000000)),
            maxUint64,
        )
        threshold.Div(threshold, big.NewInt(1000000))
        
        // 判定成功或失败
        results[i] = hashValue.Cmp(threshold) < 0
    }
    
    return results
}
```

### 概率计算
```go
func calculateStepProbability(step int, difficulty string) float64 {
    // 基于目标RTP和倍率计算
    prevMultiplier := getMultiplier(difficulty, step)
    currMultiplier := getMultiplier(difficulty, step + 1)
    
    // 概率 = 前一步倍率 ÷ 当前步倍率
    probability := prevMultiplier / currMultiplier
    
    // 应用RTP调整
    rtpAdjustment := getRTPAdjustment(difficulty)
    return probability * rtpAdjustment
}
```

## 可证明公平实现

### 核心组件
- **客户端种子**：玩家提供，确保公平性
- **服务端种子**：系统生成，游戏前哈希公开
- **Nonce**：递增计数器，确保唯一性
- **预生成结果**：游戏开始时生成所有步骤结果

### 验证流程
1. 游戏结束后公开服务端种子
2. 玩家可独立验证每步结果
3. 重新计算哈希值和概率
4. 对比实际游戏结果

### 验证 API 接口

系统提供专门的验证接口供玩家独立验证游戏结果：

**端点**: `POST /v1/fairness/chickenroad/verify`

**功能**: 使用与游戏相同的算法重新计算所有步骤结果

**请求参数**:
- `difficulty`: 难度等级（easy/medium/hard/expert）
- `clientSeed`: 客户端种子
- `serverSeed`: 服务器种子（游戏结束后公开）
- `nonce`: Nonce 值

**响应结果**:
- `stepResults[]`: 每步成功/失败的布尔数组
- `multipliers[]`: 每步对应的倍率

**验证原理**:
```
对于每一步 i（i = 0 到 maxSteps-1）：
  1. 组合种子：seedStr = "serverSeed:clientSeed:nonce:i"
  2. 计算哈希：hash = SHA256(seedStr)
  3. 转换为随机数：randomValue = hashDecimal / (2^32 - 1)
  4. 判断结果：stepResult = randomValue < survivalRate
  5. 计算倍率：从配置表中获取对应步数的倍率
```

详细使用说明请参考：[ChickenRoad WebSocket API 文档](./chickenroad-websocket-api-zh.md#11-公平性验证-api)

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/chickenroad/handler.go
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
    mu                 sync.RWMutex
}
```

### 2. 核心数据结构
```go
// 游戏实例
type GameInstance struct {
    game           *ChickenRoadGame  // 游戏逻辑
    sessionID      string            // 会话ID
    playerID       string            // 玩家ID
    userID         int64             // 用户ID
    aggregatorID   int64             // 聚合器ID
    serverSeed     *model.ServerSeed // 服务端种子
    startTime      time.Time         // 开始时间
    lastActivity   time.Time         // 最后活动时间
}

// 游戏核心
type ChickenRoadGame struct {
    // 基础信息
    RoundID        string    // 回合ID
    BetAmount      float64   // 投注金额
    Difficulty     string    // 难度
    ClientSeed     string    // 客户端种子
    serverSeed     string    // 服务端种子（私密）
    Nonce          int64     // Nonce值
    
    // 游戏状态
    Status         GameStatus // 游戏状态
    CurrentStep    int        // 当前步数
    MaxSteps       int        // 最大步数
    CompletedSteps []int      // 已完成步数
    StepResults    []bool     // 预生成结果
    
    // 赔率信息
    Multipliers    []float64  // 各步赔率
    CurrentMultiplier float64 // 当前倍数
    
    // 结果
    FinalMultiplier float64   // 最终倍数
    Payout         float64    // 奖金
    IsWin          bool       // 是否获胜
}
```

### 3. WebSocket接口

#### 游戏控制
- **PLACE_BET**：开始新游戏
  - 参数：`difficulty`、`betAmount`、通过客户端种子服务设置 `clientSeed`

- **CHICKENROAD_MOVE**：前进一步
  - 返回：是否成功、当前倍数、位置

- **CHICKENROAD_CASHOUT**：兑现
  - 返回：最终倍数、奖金

#### 会话管理
- **CHICKENROAD_CHECK_ACTIVE**：检查活跃游戏
- **CHICKENROAD_RESUME**：恢复游戏
- **CHICKENROAD_GET_STATE**：获取当前状态

### 4. 会话管理机制

#### 双索引缓存
```go
// 按用户索引
activeGamesByUser[userID][roundID] = gameInstance

// 按回合索引
activeGamesByRound[roundID] = gameInstance
```

#### 状态持久化
```go
// 序列化游戏数据
gameData := map[string]interface{}{
    "difficulty":      game.Difficulty,
    "current_step":    game.CurrentStep,
    "completed_steps": game.CompletedSteps,
    "step_results":    game.StepResults,
    "multipliers":     game.Multipliers,
    "client_seed":     game.ClientSeed,
}
```

#### 自动清理
- 超时时间：10分钟无活动
- 清理策略：自动兑现并结算
- 清理频率：每5分钟执行一次

## 前端实现建议

### 1. 界面设计

#### 游戏场景
- **道路布局**：
  - 垂直道路，分段显示
  - 每段代表一步
  - 当前位置高亮

- **小鸡角色**：
  - 动态表情（紧张、开心等）
  - 行走动画
  - 失败时的特效

#### 信息面板
- **实时数据**：
  - 当前步数/最大步数
  - 当前倍数
  - 潜在奖金
  - 下一步倍数

- **难度指示器**：
  - 颜色编码（绿、黄、橙、红）
  - 风险等级标识

### 2. 交互设计

#### 控制按钮
- **前进按钮**：
  - 大型醒目按钮
  - 显示下一步倍数
  - 风险提示

- **兑现按钮**：
  - 显示当前奖金
  - 脉冲动画吸引注意
  - 确认对话框

#### 自动游戏
- 设置目标步数
- 自动前进模式
- 失败后策略选择

### 3. 动画效果
- **前进动画**：小鸡跳跃前进
- **成功动画**：金币增长效果
- **失败动画**：碰撞或跌倒效果
- **兑现动画**：庆祝特效

### 4. 历史记录
- 路径可视化
- 每步倍数变化
- 最终结果统计
- 公平性验证数据

## 性能优化

### 1. 内存管理
- 使用对象池复用游戏实例
- 定期清理过期会话
- 限制并发游戏数量

### 2. 并发控制
- 读写锁保护共享状态
- 原子操作更新计数器
- 避免锁竞争热点

### 3. 数据库优化
- 批量写入游戏结果
- 索引优化查询性能
- 异步持久化非关键数据

## 安全考虑

1. **防作弊机制**：
   - 预生成所有结果，防止中途修改
   - 服务端验证所有操作
   - 限制操作频率

2. **并发安全**：
   - 单用户单游戏限制
   - 状态机严格校验
   - 原子操作保证一致性

3. **数据保护**：
   - 服务端种子加密存储
   - 敏感数据不暴露给客户端
   - 完整的审计日志