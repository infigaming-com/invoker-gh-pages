# Keno（基诺）游戏详细设计文档 ✅ 已实现

## 游戏规则

Keno是一种类似彩票的即时游戏：

1. **数字选择**：玩家从1-40的数字池中选择1-10个数字（称为spots）
2. **系统开奖**：系统随机抽取10个中奖数字
3. **结果判定**：根据玩家选中的数字与开奖数字的匹配数量决定赔率
4. **快速选号**：支持系统自动随机选择数字
5. **难度模式**：支持4种难度模式（low、classic、medium、high），不同难度有不同的赔率表

## 难度模式与赔率

Keno支持4种难度模式，每种模式有不同的风险与回报特性：

1. **Low（低风险）**：较高的中奖概率，但最高赔率较低
2. **Classic（经典）**：平衡的风险与回报（默认模式）
3. **Medium（中等风险）**：中等的风险与回报
4. **High（高风险）**：较低的中奖概率，但最高赔率更高

**赔率表示例（选择5个数字，Classic难度）**：

| 匹配数量 | Low | Classic | Medium | High |
|---------|-----|---------|--------|------|
| 0 | 0 | 0 | 0 | 0 |
| 1 | 0 | 0 | 0 | 0 |
| 2 | 0 | 0 | 0 | 0 |
| 3 | 0.5× | 1× | 1.5× | 2.5× |
| 4 | 5× | 10× | 25× | 50× |
| 5 | 50× | 100× | 250× | 500× |

**完整赔率表示例（选择10个数字，Classic难度）**：
| 匹配数量 | 赔率 |
|---------|------|
| 0 | 0 |
| 1 | 0 |
| 2 | 0 |
| 3 | 0 |
| 4 | 0 |
| 5 | 3× |
| 6 | 10× |
| 7 | 50× |
| 8 | 250× |
| 9 | 1000× |
| 10 | 10000× |

**难度特性对比**：
| 难度 | 特点 | 适合玩家 | 最高赔率 |
|------|------|---------|----------|
| Low | 高频小奖 | 保守型 | 较低 |
| Classic | 平衡模式 | 大众型 | 中等 |
| Medium | 中等风险 | 进取型 | 较高 |
| High | 高风险高回报 | 冒险型 | 最高 |

## 概率计算公式

匹配k个数字的概率计算：

```
P(X = k) = C(m, k) × C(40-m, n-k) / C(40, n)
```

其中：
- m = 玩家选择的数字数量
- n = 10（系统抽取的数字数量）
- k = 匹配的数字数量
- C(n, k) = 组合数（从n个中选k个）

## 随机数生成算法

使用可证明公平的Fisher-Yates洗牌算法从1-40中抽取10个不重复数字：

```go
func DrawKenoNumbers(clientSeed, serverSeed string, nonce int64) []int {
    // 1. 生成种子哈希
    seedStr := fmt.Sprintf("%s:%s:%d:keno", clientSeed, serverSeed, nonce)
    hash := sha256.Sum256([]byte(seedStr))
    
    // 2. 初始化数字池（1-40）
    numbers := make([]int, 40)
    for i := 0; i < 40; i++ {
        numbers[i] = i + 1
    }
    
    // 3. Fisher-Yates洗牌抽取前10个
    for i := 0; i < 10; i++ {
        // 使用哈希生成随机索引
        hashOffset := i * 4
        randomBytes := hash[hashOffset:hashOffset+4]
        randomIndex := binary.BigEndian.Uint32(randomBytes) % uint32(40-i)
        
        // 交换
        numbers[i], numbers[i+int(randomIndex)] = numbers[i+int(randomIndex)], numbers[i]
    }
    
    // 4. 返回前10个数字并排序
    result := numbers[:10]
    sort.Ints(result)
    return result
}
```

## 游戏流程

1. **玩家选择数字**
   - 手动选择1-10个数字
   - 或使用快速选号功能

2. **下注**
   - 验证选择的数字数量（1-10个）
   - 验证数字范围（1-40）
   - 验证下注金额
   - 验证难度模式（low/classic/medium/high）

3. **开奖**
   - 使用可证明公平算法生成10个随机数字
   - 计算匹配数量

4. **结算**
   - 根据选择的难度模式和匹配数量查找赔率
   - 计算奖金（下注金额 × 赔率）
   - 更新余额

## 前端实现建议

1. **数字选择界面**
   - 5×8的数字网格（1-40）
   - 显示已选择的数字数量
   - 难度选择器（Low/Classic/Medium/High）
   - 快速选号按钮

2. **赔率显示**
   - 根据选择的数字数量和难度动态显示赔率表
   - 高亮显示当前选择对应的所有可能赔率
   - 显示不同难度的对比

3. **开奖动画**
   - 逐个展示10个开奖数字
   - 匹配的数字特殊高亮
   - 显示最终匹配数量和赔率
   - 显示使用的难度模式

4. **历史记录**
   - 显示玩家选择的数字
   - 显示开奖数字
   - 显示匹配情况和赔率

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/keno_service.go
type KenoService struct {
    gameRepo     GameRepository
    userRepo     UserRepository 
    seedService  ServerSeedService
    engineConfig *engine.Config
}
```

### 2. 核心数据结构
```go
// api/game/v1/game_params.proto
message KenoParams {
    repeated int32 selected_numbers = 1 [json_name = "selectedNumbers"];  // 玩家选择的数字（1-10个，范围1-40）
    string difficulty = 2;  // 难度模式: "low", "classic", "medium", "high"
}

// api/game/v1/types.proto
message KenoOutcome {
    repeated int32 selected_numbers = 1 [json_name = "selectedNumbers"];  // 玩家选择的数字
    repeated int32 drawn_numbers = 2 [json_name = "drawnNumbers"];     // 系统开出的10个数字
    repeated int32 matched_numbers = 3 [json_name = "matchedNumbers"];   // 匹配的数字
    int32 match_count = 4 [json_name = "matchCount"];                    // 匹配数量
    int32 spots_count = 5 [json_name = "spotsCount"];                    // 选择数量
    string multiplier = 6;                                               // 赔率倍数
    string difficulty = 7;                                               // 使用的难度模式
}
```

### 3. WebSocket接口集成
- **消息类型**: `PLACE_BET` with `kenoParams`
- **无需会话管理**: Keno是即时游戏，一次请求完成所有操作
- **RoundID生成**: 使用Sony Flake ID生成器，纯数字格式

### 4. 验证规则
```go
func ValidateKenoParams(params *v1.KenoParams) error {
    // 数量验证：1-10个
    if len(params.SelectedNumbers) < 1 || len(params.SelectedNumbers) > 10 {
        return ErrInvalidNumberCount
    }
    
    // 范围验证：1-40
    for _, num := range params.SelectedNumbers {
        if num < 1 || num > 40 {
            return ErrInvalidNumberRange
        }
    }
    
    // 重复验证
    seen := make(map[int32]bool)
    for _, num := range params.SelectedNumbers {
        if seen[num] {
            return ErrDuplicateNumbers
        }
        seen[num] = true
    }
    
    // 难度验证（可选，默认为classic）
    if params.Difficulty != "" {
        validDifficulties := map[string]bool{
            "low": true, "classic": true, 
            "medium": true, "high": true,
        }
        if !validDifficulties[params.Difficulty] {
            return ErrInvalidDifficulty
        }
    }
    
    return nil
}
```

### 5. 性能优化
- **预计算赔率表**: 启动时加载所有赔率组合到内存
- **批量处理**: 支持并发处理多个Keno投注
- **缓存优化**: 使用LRU缓存最近的游戏结果