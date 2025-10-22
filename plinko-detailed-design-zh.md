# Plinko（弹珠台）游戏详细设计文档 ✅ 已实现

## 游戏概述

Plinko 是一个经典的弹珠掉落游戏，小球从顶部释放，通过钉桩阵列向下弹跳，最终落入底部不同槽位获得相应倍率奖励。

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