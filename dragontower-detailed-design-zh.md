# DragonTower（龙塔）游戏详细设计文档 ✅ 已实现

## 配置文件

### 游戏配置 (`configs/games/dragontower.json`)
```json
{
  "gameId": "inhousegame:dragontower",
  "status": "active",
  "oddsType": "formula",
  "rtp": 97.0,
  "gameParameters": {
    "defaultDifficulty": "medium",
    "greedPenalty": 0.15
  }
}
```

### 配置说明
- **gameId**: 游戏唯一标识符，格式为 `"inhousegame:dragontower"`
- **status**: 游戏状态（`active` 启用，`disabled` 禁用）
- **oddsType**: 赔率类型，`formula` 表示公式驱动（赔率通过数学公式实时计算）
- **rtp**: 基础理论回报率（97%），会被 GreedPenalty 动态调整
- **gameParameters**: 游戏核心参数
  - **defaultDifficulty**: 默认难度（`medium`）
  - **greedPenalty**: 贪婪惩罚系数（0.15），层数越高有效 RTP 越低

### 配置加载
- **加载时机**: 服务器启动时，GameRegistry 自动从 `configs/games/` 目录加载所有 `.json` 配置文件
- **使用方式**: 游戏服务通过 `gameRegistry.GetProtoConfig(operatorID, "inhousegame:dragontower")` 获取配置
- **Operator 覆盖**: 支持 Operator 级别的 RTP 和 gameParameters 覆盖
- **投注限额**: 从 `configs/currencies.json` 自动生成 BetInfo

## 游戏概述

DragonTower 是一个单人会话型风险博弈游戏，玩家从塔底逐层向上攀登，每层进行概率判定。成功则进入下一层并获得更高倍率，失败则损失全部投注。玩家可在任意时刻选择兑现当前倍率。

## 游戏规则

### 基本玩法
1. **选择配置**：
   - 难度档位：Easy / Medium / Hard / Expert / Master（5 档）
   - 最高层数：固定 9 层
   - 难度一旦选定，游戏过程中不可更改

2. **游戏流程**：
   - 玩家下注并选择难度
   - 从第 0 层（塔底）开始攀登
   - 每层进行一次概率判定
   - 成功：进入下一层，倍率递增
   - 失败：游戏结束，损失全部投注
   - 可随时选择 Cash Out 兑现当前倍率
   - 成功通过第 9 层自动结算获得最大奖励

3. **游戏状态**：
   - `ready` - 准备中
   - `playing` - 游戏进行中
   - `finished` - 游戏结束（失败或到达顶层）
   - `cashed_out` - 主动提现

### 难度配置

| 难度 | 代码 | 列数 | 单层成功率 | 描述 |
|------|------|------|------------|------|
| 简单 | easy | 4 | 75% (3/4) | 低风险，适合新手 |
| 中等 | medium | 3 | 66.67% (2/3) | 平衡风险与收益 |
| 困难 | hard | 2 | 50% (1/2) | 高风险高回报 |
| 专家 | expert | 3 | 33.33% (1/3) | 极高风险 |
| 大师 | master | 4 | 25% (1/4) | 最高风险，最高回报 |

单层成功率与层数无关，每层的边际成功率恒等于难度成功率 p。

## 赔率计算公式

### 核心算法

DragonTower 使用公式驱动的赔率计算，引入 GreedPenalty 机制使有效 RTP 随层数递减：

```go
func (g *Game) GetEffectiveRTP(layer int) float64 {
    normalizedLayer := float64(layer) / float64(g.MaxLayers)
    return g.RTP * (1.0 - g.GreedPenalty * normalizedLayer * normalizedLayer)
}

func (g *Game) CalculateMultiplierForLayer(layer int) float64 {
    if layer == 0 {
        return 1.0
    }
    effectiveRTP := g.GetEffectiveRTP(layer)
    return effectiveRTP / math.Pow(g.SuccessRate, float64(layer))
}
```

### 关键参数
- **基础 RTP**: 97%（配置可调）
- **GreedPenalty**: 0.15（配置可调）
- **有效 RTP 公式**: `RTP × (1 - greedPenalty × (n / maxLayers)²)`
- **倍率公式**: `有效RTP(n) / 成功率^n`
- **倍率精度**: 8 位小数，无四舍五入
- **最高层数**: 9 层

### GreedPenalty 机制

GreedPenalty 使层数越高的有效 RTP 越低，激励玩家适时提现：

| 层数 | normalizedLayer | 有效 RTP |
|------|----------------|----------|
| 1 | 0.111 | 96.82% |
| 3 | 0.333 | 95.38% |
| 5 | 0.556 | 92.51% |
| 7 | 0.778 | 88.20% |
| 9 | 1.000 | 82.45% |

### 各难度倍率表

#### Easy 难度 (p=0.75)
| 层数 | 有效 RTP | 倍率 |
|------|----------|------|
| 1 | 96.82% | 1.29x |
| 2 | 96.28% | 1.71x |
| 3 | 95.38% | 2.26x |
| 4 | 94.13% | 2.97x |
| 5 | 92.51% | 3.90x |
| 6 | 90.53% | 5.09x |
| 7 | 88.20% | 6.61x |
| 8 | 85.50% | 8.54x |
| 9 | 82.45% | 10.98x |

#### Medium 难度 (p≈0.6667)
| 层数 | 有效 RTP | 倍率 |
|------|----------|------|
| 1 | 96.82% | 1.45x |
| 2 | 96.28% | 2.17x |
| 3 | 95.38% | 3.22x |
| 4 | 94.13% | 4.76x |
| 5 | 92.51% | 7.02x |
| 6 | 90.53% | 10.31x |
| 7 | 88.20% | 15.07x |
| 8 | 85.50% | 21.91x |
| 9 | 82.45% | 31.70x |

#### Hard 难度 (p=0.5)
| 层数 | 有效 RTP | 倍率 |
|------|----------|------|
| 1 | 96.82% | 1.94x |
| 2 | 96.28% | 3.85x |
| 3 | 95.38% | 7.63x |
| 4 | 94.13% | 15.06x |
| 5 | 92.51% | 29.60x |
| 6 | 90.53% | 57.94x |
| 7 | 88.20% | 112.89x |
| 8 | 85.50% | 218.89x |
| 9 | 82.45% | 422.19x |

#### Expert 难度 (p≈0.3333)
| 层数 | 有效 RTP | 倍率 |
|------|----------|------|
| 1 | 96.82% | 2.90x |
| 2 | 96.28% | 8.67x |
| 3 | 95.38% | 25.75x |
| 4 | 94.13% | 76.25x |
| 5 | 92.51% | 224.82x |
| 6 | 90.53% | 660.00x |
| 7 | 88.20% | 1929.04x |
| 8 | 85.50% | 5610.40x |
| 9 | 82.45% | 16228.24x |

#### Master 难度 (p=0.25)
| 层数 | 有效 RTP | 倍率 |
|------|----------|------|
| 1 | 96.82% | 3.87x |
| 2 | 96.28% | 15.41x |
| 3 | 95.38% | 61.05x |
| 4 | 94.13% | 240.96x |
| 5 | 92.51% | 947.30x |
| 6 | 90.53% | 3708.36x |
| 7 | 88.20% | 14449.59x |
| 8 | 85.50% | 56035.71x |
| 9 | 82.45% | 216102.41x |

## 可证明公平实现

### 随机数生成算法
```go
func generateRandomForLayer(serverSeed, clientSeed string, nonce int64, layer int) float64 {
    seedStr := fmt.Sprintf("%s:%s:%d:%d", serverSeed, clientSeed, nonce, layer)

    hash := sha256.Sum256([]byte(seedStr))
    hashHex := hex.EncodeToString(hash[:])
    hashFirst8 := hashHex[:8]

    hashDecimal := new(big.Int)
    hashDecimal.SetString(hashFirst8, 16)

    // 除以 0x100000000 (2^32) 映射到 [0, 1)
    divisor := new(big.Int).SetInt64(0x100000000)
    result := new(big.Float).SetInt(hashDecimal)
    result.Quo(result, new(big.Float).SetInt(divisor))

    floatResult, _ := result.Float64()
    return floatResult
}

func checkLayerResult(randomValue float64, successRate float64) bool {
    return randomValue < successRate
}
```

### 预生成机制

游戏开始时，使用种子一次性预生成所有 9 层的结果：

```go
func (g *Game) PreGenerateAllLayers() {
    g.LayerResults = make([]bool, g.MaxLayers)
    for i := 0; i < g.MaxLayers; i++ {
        seedStr := BuildIndexedSeed(g.ServerSeed, g.ClientSeed, g.Nonce, i)
        randomValue := Float64(seedStr)
        g.LayerResults[i] = randomValue < g.SuccessRate
    }
}
```

### 公平性保障
- **种子顺序**: `serverSeed:clientSeed:nonce:layerIndex`
- **客户端种子**: 玩家提供，确保随机性不受服务器控制
- **服务器种子**: 游戏开始前生成，结束后公开供验证
- **Nonce**: 递增计数器，确保每局唯一性
- **层索引**: 确保每层的随机值独立

### 验证流程
1. 玩家获取游戏记录（包含服务器种子明文）
2. 使用相同算法重新生成每层的随机值
3. 对比随机值与成功率，验证每层判定结果
4. 计算倍率并验证兑现金额

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/dragontower/service.go
type Service struct {
    *games.SessionGameBase[*dragontower.Game, *v1.DragonTowerPlaceBetRequest]
    gameRegistry *provider.GameRegistry
}
```

### 2. 核心数据结构
```go
type Game struct {
    game.SessionGame

    CurrentMultiplier float64
    Difficulty        string
    CurrentLayer      int
    MaxLayers         int
    CompletedLayers   []int
    Survived          bool
    CashedOut         bool
    NextMultiplier    float64

    SuccessRate       float64  // 从难度配置加载，不序列化
    GreedPenalty      float64  // 从游戏配置加载，不序列化
    LayerResults      []bool   // 预生成结果，不序列化
}
```

`game.SessionGame` 内嵌 `game.Game`，包含 `BetAmount`（decimal.Decimal）、`ClientSeed`、`ServerSeed`、`Nonce`、`FinalPayout`（decimal.Decimal）、`RTP`、`IsWin`、`IsDemo` 等基础字段。

### 3. WebSocket 接口

#### 游戏控制
- **placeBet**: 开始新游戏（选择难度和投注金额）
- **climb**: 攀登一层
- **cashOut**: 兑现
- **autoPlay**: 自动攀登到目标层

#### 会话管理
- **checkActive**: 检查活跃游戏

### 4. 游戏模式

#### 手动模式（climb）
逐层操作，玩家手动决定每层是继续攀登还是兑现。

#### 自动模式（autoPlay）
一次性 RPC 调用完成下注 + 攀登到目标层。达到目标层自动 CashOut，中途失败立即结束。不支持中途停止。

### 5. 会话管理

#### 状态持久化
- 游戏状态序列化存储到 `GameSession` 表
- 断线重连时通过 `Restore()` 恢复运行时数据（SuccessRate、LayerResults、NextMultiplier）

#### 生命周期
1. **创建**: PlaceBet 时创建新会话
2. **持久化**: 每次 climb 后更新 session 数据
3. **结束**: 失败/到达顶层/CashOut 时调用 ProcessGameEnd

## 与 ChickenRoad 的对比

| 特性 | ChickenRoad | DragonTower |
|------|-------------|-------------|
| 游戏类型 | Session 游戏 | Session 游戏 |
| 赔率类型 | table（表驱动） | formula（公式驱动） |
| 最大步数/层数 | 不同难度不同 | 固定 9 层 |
| 难度影响 | 最大步数和赔率表 | 单层成功率 |
| 核心操作 | move | climb |
| 倍率计算 | 查表 | 公式 + GreedPenalty |
| RTP | 98% | 97%（有效 RTP 随层数递减） |
| 倍率精度 | 8 位小数 | 8 位小数 |
| 自动模式 | autoPlay | autoPlay |
| 随机生成 | 预生成所有步骤 | 预生成所有层级 |
