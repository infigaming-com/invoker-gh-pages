# HiLo（高低牌）游戏详细设计文档 ✅ 已实现

## 配置文件

### 游戏配置 (`configs/games/hilo.json`)
```json
{
  "gameId": "inhousegame:hilo",
  "status": "active",
  "oddsType": "formula",
  "rtp": 96.0,
  "gameParameters": {}
}
```

### 配置说明
- **gameId**: 游戏唯一标识符，格式为 `"inhousegame:hilo"`
- **status**: 游戏状态（`active` 启用，`disabled` 禁用）
- **oddsType**: 赔率类型，`formula` 表示公式驱动（赔率通过数学公式实时计算）
- **rtp**: 理论回报率（96%）
- **gameParameters**: 无额外参数

### 配置加载
- **加载时机**: 服务器启动时，GameRegistry 自动从 `configs/games/` 目录加载所有 `.json` 配置文件
- **使用方式**: 游戏服务通过 `gameRegistry.GetGame(operatorID, "inhousegame:hilo")` 获取配置
- **Operator 覆盖**: 支持 Operator 级别的 RTP 覆盖
- **投注限额**: 从 `configs/currencies.json` 自动生成 BetInfo

## 游戏概述

HiLo 是一个经典的卡牌预测游戏，玩家需要预测下一张牌比当前牌高还是低。每次猜对后倍率累积增长，玩家可随时选择提现或继续挑战更高倍率。

## 游戏规则

### 基本玩法
1. **开始游戏**：
   - 设置投注金额和客户端种子
   - 系统发第一张牌
   - 游戏进入 playing 状态

2. **做出选择**：
   - **higher**: 预测下张牌 >= 当前牌
   - **lower**: 预测下张牌 <= 当前牌
   - **skip**: 跳过换牌（不影响倍率，最多 50 次）

3. **判定结果**：
   - 猜对：倍率累积，可继续或提现
   - 猜错：游戏结束，失去投注

4. **提现**：
   - 至少猜对一次后可提现
   - 最终赔付 = 投注金额 × 累积倍率

### 牌面值
A=1, 2-10, J=11, Q=12, K=13

### 游戏状态
- `ready` - 准备中
- `playing` - 游戏进行中
- `finished` - 游戏结束（猜错）
- `cashed_out` - 主动提现

### 判定规则

使用**宽松判定**（包含相等情况）：

- Higher: `nextCard >= currentCard`
- Lower: `nextCard <= currentCard`

当下张牌等于当前牌时，Higher 和 Lower 都算赢。因此概率之和 > 100%，这是对玩家有利的设计。

## 赔率计算公式

### 核心算法

```go
func (g *Game) calculateProbability(choice string) float64 {
    switch choice {
    case "higher":
        return float64(14-g.CurrentCard) / 13.0
    case "lower":
        return float64(g.CurrentCard) / 13.0
    default:
        return 0
    }
}

func (g *Game) calculateMultiplier(probability float64) float64 {
    multiplier := g.RTP / probability
    return math.Round(multiplier*10000) / 10000  // 保留 4 位小数
}
```

### 关键参数
- **RTP**: 96%（配置可调）
- **单步倍率精度**: 4 位小数（`math.Round`）
- **累积倍率**: 各步倍率相乘，API 返回 8 位小数
- **牌值范围**: 1-13（A-K）

### 概率与倍率表

| 当前牌 | Higher 概率 | Higher 倍率 | Lower 概率 | Lower 倍率 |
|--------|------------|------------|-----------|-----------|
| A (1) | 13/13 (100%) | 0.96x | 1/13 (7.7%) | 12.48x |
| 2 | 12/13 (92.3%) | 1.04x | 2/13 (15.4%) | 6.24x |
| 3 | 11/13 (84.6%) | 1.1345x | 3/13 (23.1%) | 4.16x |
| 4 | 10/13 (76.9%) | 1.248x | 4/13 (30.8%) | 3.12x |
| 5 | 9/13 (69.2%) | 1.3867x | 5/13 (38.5%) | 2.496x |
| 6 | 8/13 (61.5%) | 1.56x | 6/13 (46.2%) | 2.08x |
| 7 | 7/13 (53.8%) | 1.7829x | 7/13 (53.8%) | 1.7829x |
| 8 | 6/13 (46.2%) | 2.08x | 8/13 (61.5%) | 1.56x |
| 9 | 5/13 (38.5%) | 2.496x | 9/13 (69.2%) | 1.3867x |
| 10 | 4/13 (30.8%) | 3.12x | 10/13 (76.9%) | 1.248x |
| J (11) | 3/13 (23.1%) | 4.16x | 11/13 (84.6%) | 1.1345x |
| Q (12) | 2/13 (15.4%) | 6.24x | 12/13 (92.3%) | 1.04x |
| K (13) | 1/13 (7.7%) | 12.48x | 13/13 (100%) | 0.96x |

**注意**：
- A 选 Higher 和 K 选 Lower 必赢，但倍率为 0.96x（亏损），不存在"无法选择"的情况
- 概率公式包含相等情况，所以 Higher+Lower 概率之和 > 100%

### Skip 机制
- 最大跳过次数: 50
- 跳过不影响累积倍率
- 跳过时换一张新牌，可用于等待更有利的牌面
- 任何时候都可以跳过（不限于首次猜测前）

## 可证明公平实现

### 牌序生成算法

采用动态批量生成策略（初始 20 张，按需扩展 20 张）：

```go
func (g *Game) generateCards(count int) {
    startIndex := len(g.cardSequence)
    for i := 0; i < count; i++ {
        seedStr := random.BuildIndexedSeed(g.ServerSeed, g.ClientSeed, g.Nonce, startIndex+i)
        cardValue := random.Int64n(seedStr, 13) + 1  // 1-13
        g.cardSequence = append(g.cardSequence, int(cardValue))
    }
}
```

### 单张牌生成过程

```
1. 组合种子：seedStr = "serverSeed:clientSeed:nonce:cardIndex"
2. 计算哈希：hash = SHA256(seedStr)
3. 取前 8 位十六进制：hashFirst8 = hash[:8]
4. 转换为整数：hashValue = parseInt(hashFirst8, 16)
5. 计算牌值：card = (hashValue % 13) + 1
```

### 公平性保障
- **种子顺序**: `serverSeed:clientSeed:nonce:cardIndex`
- **客户端种子**: 玩家提供，确保随机性不受服务器控制
- **服务器种子**: 游戏开始前生成，结束后公开供验证
- **Nonce**: 递增计数器，确保每局唯一性
- **牌索引**: 确保每张牌的随机值独立
- **动态生成**: 牌序按需生成，但结果由种子决定，具有确定性

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/hilo/service.go
type Service struct {
    *games.SessionGameBase[*hilogame.Game, *v1.HiloPlaceBetRequest]
    gameRegistry *provider.GameRegistry
}
```

### 2. 核心数据结构
```go
type Game struct {
    game.SessionGame

    CurrentCard   int
    CardHistory   []int
    nextCardIndex int    // 不序列化
    cardSequence  []int  // 不序列化

    CurrentMultiplier float64
    MultiplierHistory []float64
    NextMultiplier    float64   // 不序列化，由 updateNextFields 计算
    NextProbability   float64   // 不序列化，由 updateNextFields 计算

    CanCashout bool
    GuessCount int
    SkipCount  int
}
```

`game.SessionGame` 内嵌 `game.Game`，包含 `BetAmount`（decimal.Decimal）、`ClientSeed`、`ServerSeed`、`Nonce`、`FinalPayout`（decimal.Decimal）、`RTP`、`IsWin`、`IsDemo` 等基础字段。

### 3. WebSocket 接口

#### 游戏控制
- **placeBet**: 开始新游戏
- **choice**: 做出选择（higher/lower/skip）
- **cashOut**: 兑现

#### 会话管理
- **checkActive**: 检查活跃游戏

### 4. NextMultiplier 逻辑

`nextMultiplier` 字段取 Higher 和 Lower 中概率更高的那个选项对应的累积倍率：

```go
func (g *Game) updateNextFields() {
    higherProb := g.calculateProbability("higher")
    lowerProb := g.calculateProbability("lower")

    if higherProb >= lowerProb {
        g.NextProbability = higherProb
        g.NextMultiplier = g.CurrentMultiplier * g.calculateMultiplier(higherProb)
    } else {
        g.NextProbability = lowerProb
        g.NextMultiplier = g.CurrentMultiplier * g.calculateMultiplier(lowerProb)
    }
}
```

### 5. 会话管理

#### 状态恢复
断线重连时，通过 `Restore()` 重新生成牌序并恢复运行时数据：
```go
func (g *Game) Restore() {
    g.cardSequence = nil
    g.generateCards(len(g.CardHistory) + expandBatchSize)
    g.nextCardIndex = len(g.CardHistory)
    g.updateNextFields()
}
```

#### 生命周期
1. **创建**: PlaceBet 时创建新会话，发第一张牌
2. **持久化**: 每次 choice 后更新 session 数据
3. **结束**: 猜错/CashOut 时调用 ProcessGameEnd

## 与 ChickenRoad/DragonTower 的对比

| 特性 | HiLo | ChickenRoad | DragonTower |
|------|------|-------------|-------------|
| 游戏类型 | Session 游戏 | Session 游戏 | Session 游戏 |
| 赔率类型 | formula | table | formula |
| 核心机制 | 预测牌高低 | 前进闯关 | 攀登闯关 |
| 判定依据 | 牌面比较 | 随机值 < 存活率 | 随机值 < 成功率 |
| 倍率计算 | RTP / 概率 | 查表 | 公式 + GreedPenalty |
| RTP | 96% | 98% | 97% |
| Skip 机制 | 有（最多 50 次） | 无 | 无 |
| 倍率精度 | 单步 4 位，显示 8 位 | 8 位 | 8 位 |
| 随机生成 | 动态批量（20 张/批） | 预生成所有步骤 | 预生成所有层级 |
