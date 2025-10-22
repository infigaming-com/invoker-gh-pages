# DragonTower（龙之塔）游戏详细设计文档

## 状态标记
状态：✅ 已实现

## 游戏概述

DragonTower（龙之塔）是一个单人会话型风险博弈游戏，玩家从塔底逐层向上闯关，每层进行概率判定。成功则进入下一层并获得更高倍率，失败则损失全部投注。玩家可在任意时刻选择兑现当前倍率。

## 游戏规则

### 基本玩法
1. **选择配置**：
   - 难度档位：Easy / Medium / Hard / Expert / Master（5档）
   - 最高层数：固定9层
   - 难度一旦选定，游戏过程中不可更改

2. **游戏流程**：
   - 玩家下注并选择难度
   - 从第0层（塔底）开始闯关
   - 每层进行一次概率判定（伯努利试验）
   - 成功：进入下一层，倍率递增
   - 失败：游戏结束，损失全部投注
   - 可随时选择 Cash Out 兑现当前倍率
   - 成功通过第9层获得最大奖励

3. **游戏状态**：
   - `ready` - 准备中
   - `playing` - 游戏进行中
   - `finished` - 游戏结束（失败）
   - `cashed_out` - 主动提现

### 难度配置

| 难度 | 英文代码 | 单层成功率 | 描述 |
|------|----------|------------|------|
| 简单 | easy | 75% (3/4) | 适合新手，风险较低 |
| 中等 | medium | 66.67% (2/3) | 平衡风险与收益 |
| 困难 | hard | 50% (1/2) | 高风险高回报 |
| 专家 | expert | 33.33% (1/3) | 极高风险 |
| 大师 | master | 25% (1/4) | 最高风险，最高回报 |

**注意**：单层成功率与层数无关，每层的边际成功率恒等于难度成功率 p

## 赔率计算公式

### 核心算法
```
累计生存率: S(n) = p^n
原始倍率: M_raw(n) = 0.98 / p^n
显示倍率: M(n) = RoundHalfUp(M_raw(n), 2)
兑现金额 = 下注金额 × M(n)
```

### 关键参数
- **RTP（理论回报率）**：98%
- **倍率精度**：四舍五入到2位小数
- **最高层数**：9层

### 各难度倍率表

#### Easy 难度 (p=0.75)
| 层数 | 累计生存率 | 显示倍率 |
|------|-----------|----------|
| 1 | 0.75000 | 1.31 |
| 2 | 0.56250 | 1.74 |
| 3 | 0.42188 | 2.32 |
| 4 | 0.31641 | 3.10 |
| 5 | 0.23730 | 4.13 |
| 6 | 0.17798 | 5.51 |
| 7 | 0.13348 | 7.34 |
| 8 | 0.10011 | 9.79 |
| 9 | 0.07508 | 13.05 |

#### Medium 难度 (p≈0.6667)
| 层数 | 累计生存率 | 显示倍率 |
|------|-----------|----------|
| 1 | 0.66667 | 1.47 |
| 2 | 0.44444 | 2.21 |
| 3 | 0.29630 | 3.31 |
| 4 | 0.19753 | 4.96 |
| 5 | 0.13169 | 7.44 |
| 6 | 0.08779 | 11.16 |
| 7 | 0.05853 | 16.74 |
| 8 | 0.03902 | 25.12 |
| 9 | 0.02601 | 37.67 |

#### Hard 难度 (p=0.5)
| 层数 | 累计生存率 | 显示倍率 |
|------|-----------|----------|
| 1 | 0.50000 | 1.96 |
| 2 | 0.25000 | 3.92 |
| 3 | 0.12500 | 7.84 |
| 4 | 0.06250 | 15.68 |
| 5 | 0.03125 | 31.36 |
| 6 | 0.01563 | 62.72 |
| 7 | 0.00781 | 125.44 |
| 8 | 0.00391 | 250.88 |
| 9 | 0.00195 | 501.76 |

#### Expert 难度 (p≈0.3333)
| 层数 | 累计生存率 | 显示倍率 |
|------|-----------|----------|
| 1 | 0.33333 | 2.94 |
| 2 | 0.11111 | 8.82 |
| 3 | 0.03704 | 26.46 |
| 4 | 0.01235 | 79.38 |
| 5 | 0.00412 | 238.14 |
| 6 | 0.00137 | 714.42 |
| 7 | 0.00046 | 2143.26 |
| 8 | 0.00015 | 6429.78 |
| 9 | 0.00005 | 19289.34 |

#### Master 难度 (p=0.25)
| 层数 | 累计生存率 | 显示倍率 |
|------|-----------|----------|
| 1 | 0.25000 | 3.92 |
| 2 | 0.06250 | 15.68 |
| 3 | 0.01563 | 62.72 |
| 4 | 0.00391 | 250.88 |
| 5 | 0.00098 | 1003.52 |
| 6 | 0.00024 | 4014.08 |
| 7 | 0.00006 | 16056.32 |
| 8 | 0.00002 | 64225.28 |
| 9 | 0.00000 | 256901.12 |

## 可证明公平实现

### 随机数生成算法
```go
func generateRandomForLayer(clientSeed, serverSeed string, nonce int64, layer int) float64 {
    // 生成种子字符串
    seedStr := fmt.Sprintf("%s:%s:%d:%d", clientSeed, serverSeed, nonce, layer)

    // 计算 SHA256 哈希
    hash := sha256.Sum256([]byte(seedStr))
    hashHex := hex.EncodeToString(hash[:])

    // 取前8位转换为整数
    hashValue := new(big.Int)
    hashValue.SetString(hashHex[:8], 16)

    // 映射到 [0,1) 区间
    max32Bit := new(big.Int).SetInt64(0xFFFFFFFF)
    result := new(big.Float).SetInt(hashValue)
    divisor := new(big.Float).SetInt(max32Bit)
    result.Quo(result, divisor)

    floatResult, _ := result.Float64()
    return floatResult
}

func checkLayerResult(randomValue float64, successRate float64) bool {
    return randomValue < successRate
}
```

### 公平性保障
- **客户端种子**：玩家提供（8-256字符），确保随机性不受服务器控制
- **服务器种子**：游戏开始前生成，结束后公开供验证
- **Nonce**：局内唯一标识
- **层索引**：确保每层的随机值独立
- **哈希公开**：游戏开始前公开服务器种子哈希值
- **明文披露**：游戏结束后公开服务器种子明文

### 验证流程
1. 玩家获取游戏记录（包含服务器种子明文）
2. 使用相同算法重新生成每层的随机值
3. 对比随机值与成功率，验证每层判定结果
4. 计算倍率并验证兑现金额

## 游戏模式

### 手动模式
- **逐层操作**：玩家手动决定每层是继续闯关还是兑现
- **实时显示**：
  - 当前层数和成功率
  - 当前倍率和可兑现金额
  - 下一层的预期倍率
- **即时反馈**：每层结果立即展示
- **兑现控制**：玩家完全控制兑现时机

### 自动模式
- **目标设置**：预设自动兑现层数（1-9层）
- **批量执行**：系统自动执行直到目标层或失败
- **即时兑现**：达到目标层立即自动兑现
- **中途停止**：玩家可随时切换回手动模式
- **状态监控**：实时显示进度和统计信息

## 状态管理

### 游戏状态字段
```go
type DragonTowerGame struct {
    RoundID           string      // 回合ID
    BetAmount         float64     // 下注金额
    Difficulty        Difficulty  // 难度档位
    ClientSeed        string      // 客户端种子
    ServerSeed        string      // 服务器种子
    Nonce             int64       // Nonce
    Status            GameStatus  // 游戏状态
    CurrentLayer      int         // 当前层数 (0-9)
    MaxLayers         int         // 最大层数（固定9）
    CurrentMultiplier float64     // 当前倍率
    CompletedLayers   []int       // 已完成层数列表
    Survived          bool        // 是否存活
    CashedOut         bool        // 是否已兑现
    FinalPayout       float64     // 最终赔付
    LayerResults      []bool      // 每层结果（预生成）
}
```

### 状态持久化
- **会话存储**：将游戏状态序列化存储到 `GameSession` 表
- **断线重连**：基于确定性种子，确保续局结果一致
- **状态恢复**：包括难度、层数、倍率、已完成层数等

## 业务规则

### BR-001：游戏开始规则
- 玩家必须输入有效下注金额
- 玩家必须选择难度档位
- 游戏开始后，下注金额和难度不可更改
- 支持手动模式和自动模式
- 同一玩家同一时刻仅允许一局进行

### BR-002：难度与成功率规则
- Easy：成功率 75%
- Medium：成功率 66.67%
- Hard：成功率 50%
- Expert：成功率 33.33%
- Master：成功率 25%
- 单层成功率与层数无关，每层边际成功率恒等于 p

### BR-003：层级与倍率计算规则
- 累计生存率：S(n) = p^n
- 原始倍率：M_raw(n) = 0.98 / p^n
- 显示倍率：M(n) = RoundHalfUp(M_raw(n), 2)
- 兑现金额 = 下注金额 × M(n)
- 所有倍率显示和结算采用两位小数四舍五入

### BR-004：闯关判定规则
- 每层进行一次伯努利试验判定
- 成功：进入下一层，倍率更新
- 失败：游戏结束，损失全部下注金额
- 玩家可在任意时刻选择 Cash Out 兑现
- 最多可闯关9层

### BR-005：兑现（Cash Out）规则
- 手动模式：玩家任意时刻可点击 Cash Out
- 自动模式：达到预设目标层自动兑现
- 兑现金额 = 下注金额 × 当前层显示倍率 M(n)
- 兑现操作不可撤销
- 断线情况下以服务端接收时间为准

### BR-006：断线重连规则
- 状态持久化：服务端保存完整游戏状态
- 确定性续局：每层结果由固定种子生成，断线不影响结果
- 状态恢复：包括难度、层数、倍率、已完成层数等
- 并发保护：同一局仅允许一个活动会话
- 幂等处理：重复请求不会改变游戏状态

## 技术实现要点

### 1. 倍率精度处理
```go
func CalculateMultiplierForLayer(layer int, successRate float64) float64 {
    rawMultiplier := 0.98 / math.Pow(successRate, float64(layer))
    // 四舍五入到2位小数
    return math.Round(rawMultiplier * 100) / 100
}
```

### 2. 预生成层级结果
```go
func (g *DragonTowerGame) preGenerateAllLayers() {
    g.LayerResults = make([]bool, g.MaxLayers)
    successRate := getDifficultySuccessRate(g.Difficulty)

    for i := 0; i < g.MaxLayers; i++ {
        randomValue := g.generateRandomForLayer(i)
        g.LayerResults[i] = randomValue < successRate
    }
}
```

### 3. 闯关操作
```go
func (g *DragonTowerGame) ClimbLayer() (bool, error) {
    if g.Status != StatusPlaying {
        return false, fmt.Errorf("game not in playing state")
    }

    if g.CurrentLayer >= g.MaxLayers {
        return false, fmt.Errorf("already at maximum layers")
    }

    survived := g.LayerResults[g.CurrentLayer]

    if survived {
        g.CompletedLayers = append(g.CompletedLayers, g.CurrentLayer)
        g.CurrentLayer++
        g.CurrentMultiplier = g.CalculateMultiplierForLayer(g.CurrentLayer)
        g.Survived = true

        if g.CurrentLayer >= g.MaxLayers {
            g.FinalPayout = g.BetAmount * g.CurrentMultiplier
            g.Status = StatusFinished
            g.CashedOut = true
        }
    } else {
        g.Survived = false
        g.FinalPayout = 0
        g.Status = StatusFinished
        g.CurrentMultiplier = 0
    }

    return survived, nil
}
```

## 与 ChickenRoad 的对比

| 特性 | ChickenRoad | DragonTower |
|------|-------------|-------------|
| 游戏类型 | Session 游戏 | Session 游戏 |
| 最大步数/层数 | 不同难度不同（10-19步） | 固定9层 |
| 难度影响 | 最大步数 | 单层成功率 |
| 核心操作 | MoveForward | ClimbLayer |
| 倍率计算 | 基于生存率递归计算 | 直接公式 0.98/p^n |
| 倍率精度 | 4位小数 | 2位小数（四舍五入） |
| RTP | 98% | 98% |
| 随机生成 | 预生成所有步骤 | 预生成所有层级 |

## API 设计

### WebSocket 消息类型
- `PLACE_BET` - 下注开始游戏（通用适配器）
- `DRAGONTOWER_CLIMB` - 尝试闯关下一层
- `DRAGONTOWER_CASHOUT` - 兑现
- `DRAGONTOWER_CHECK_ACTIVE` - 检查活跃游戏
- `DRAGONTOWER_RESUME` - 恢复游戏
- `DRAGONTOWER_GET_STATE` - 获取游戏状态

### 响应数据格式
```json
{
    "status": "playing",
    "currentLayer": 3,
    "maxLayers": 9,
    "difficulty": "medium",
    "currentMultiplier": "3.31",
    "completedLayers": [0, 1, 2],
    "survived": true,
    "cashedOut": false,
    "betAmount": "10.00000000",
    "nextMultiplier": "4.96",
    "nextProbability": 0.6667
}
```

## 测试要点

### 单元测试
1. 倍率计算准确性验证
2. 随机数生成确定性验证
3. 可证明公平验证
4. 状态恢复验证
5. 边界条件测试

### 集成测试
1. 完整游戏流程测试
2. 断线重连测试
3. 并发安全测试
4. 不同难度档位测试
