# Roulette（轮盘）游戏详细设计文档

## 配置文件

### 游戏配置 (`configs/games/roulette.json`)
```json
{
  "gameId": "inhousegame:roulette",
  "status": "active",
  "oddsType": "formula",
  "rtp": 96,
  "gameParameters": {
    "wheelType": "european",
    "totalNumbers": 37,
    "redNumbers": [1, 3, 5, 7, 9, 12, 14, 16, 18, 19, 21, 23, 25, 27, 30, 32, 34, 36],
    "blackNumbers": [2, 4, 6, 8, 10, 11, 13, 15, 17, 20, 22, 24, 26, 28, 29, 31, 33, 35],
    "betTypes": {
      "straight": {"multiplier": 36.0, "coverage": 1},
      "split": {"multiplier": 18.0, "coverage": 2},
      "street": {"multiplier": 12.0, "coverage": 3},
      "corner": {"multiplier": 9.0, "coverage": 4},
      "sixline": {"multiplier": 6.0, "coverage": 6},
      "column": {"multiplier": 3.0, "coverage": 12},
      "dozen": {"multiplier": 3.0, "coverage": 12},
      "red": {"multiplier": 2.0, "coverage": 18},
      "black": {"multiplier": 2.0, "coverage": 18},
      "odd": {"multiplier": 2.0, "coverage": 18},
      "even": {"multiplier": 2.0, "coverage": 18},
      "low": {"multiplier": 2.0, "coverage": 18},
      "high": {"multiplier": 2.0, "coverage": 18}
    }
  }
}
```

### 配置说明
- **gameId**: 游戏唯一标识符，格式为 `"inhousegame:roulette"`
- **status**: 游戏状态（`active` 启用，`disabled` 禁用）
- **oddsType**: 赔率类型，`formula` 表示公式驱动（倍率通过 `(totalNumbers / coverage) × RTP` 实时计算）
- **rtp**: 理论回报率（96%）
- **gameParameters**: 游戏核心参数
  - **wheelType**: 轮盘类型（`european` 欧式）
  - **totalNumbers**: 总数字数（37，即 0-36）
  - **redNumbers**: 红色号码列表（18 个）
  - **blackNumbers**: 黑色号码列表（18 个）
  - **betTypes**: 投注类型配置，每种包含 `multiplier`（基础倍率）和 `coverage`（覆盖数字数量）

### 配置加载
- **加载时机**: 服务器启动时，GameRegistry 自动从 `configs/games/` 目录加载所有 `.json` 配置文件
- **倍率重算**: `BuildProtoConfig` 中根据 RTP 重新计算所有倍率：`(totalNumbers / coverage) × (rtp / 100)`
- **Operator 覆盖**: 支持 Operator 级别的 RTP 覆盖，倍率随之自动重算
- **投注限额**: 从 `configs/currencies.json` 自动生成 BetInfo

## 游戏概述

Roulette 是经典的欧式轮盘赌游戏，玩家在 0-36 共 37 个数字上选择一种或多种投注方式下注，系统生成中奖号码后结算所有投注。支持 13 种投注类型，单次请求可包含多个投注。

## 游戏规则

### 基本玩法
1. **下注**：选择一个或多个投注类型，每个投注设置金额
2. **开奖**：系统生成 0-36 的中奖号码
3. **结算**：遍历所有投注，命中的投注按对应倍率派彩

### 轮盘布局

**颜色分布**：
- **绿色 (1个)**: 0
- **红色 (18个)**: 1, 3, 5, 7, 9, 12, 14, 16, 18, 19, 21, 23, 25, 27, 30, 32, 34, 36
- **黑色 (18个)**: 2, 4, 6, 8, 10, 11, 13, 15, 17, 20, 22, 24, 26, 28, 29, 31, 33, 35

### 13 种投注类型

#### 内围投注（Inside Bets）

| 类型 | 说明 | numbers 要求 | 覆盖数 |
|------|------|-------------|--------|
| `straight` | 单号直注 | 1 个数字（0-36） | 1 |
| `split` | 两号分注 | 2 个相邻数字 | 2 |
| `street` | 三号街注 | 3 个同行数字 | 3 |
| `corner` | 四号角注 | 4 个相邻数字 | 4 |
| `sixline` | 六号线注 | 6 个两行数字 | 6 |

#### 外围投注（Outside Bets）

| 类型 | 说明 | numbers 要求 | 覆盖数 |
|------|------|-------------|--------|
| `column` | 列投注 | 1 个值（1/2/3） | 12 |
| `dozen` | 打投注 | 1 个值（1/2/3） | 12 |
| `red` | 红色投注 | 不需要 | 18 |
| `black` | 黑色投注 | 不需要 | 18 |
| `odd` | 奇数投注 | 不需要 | 18 |
| `even` | 偶数投注 | 不需要 | 18 |
| `low` | 小号投注（1-18） | 不需要 | 18 |
| `high` | 大号投注（19-36） | 不需要 | 18 |

**Column 列判定**：使用取模运算 `winningNumber % 3 == column % 3`
- 第 1 列（column=1）：1, 4, 7, 10, 13, 16, 19, 22, 25, 28, 31, 34
- 第 2 列（column=2）：2, 5, 8, 11, 14, 17, 20, 23, 26, 29, 32, 35
- 第 3 列（column=3）：3, 6, 9, 12, 15, 18, 21, 24, 27, 30, 33, 36

**Dozen 打判定**：
- 第 1 打（dozen=1）：1-12
- 第 2 打（dozen=2）：13-24
- 第 3 打（dozen=3）：25-36

**0 号特性**：column、dozen 和所有外围投注（red/black/odd/even/low/high）均不覆盖 0 号。

## 赔率计算公式

### 核心算法

```go
// Service 层 BuildProtoConfig 中动态计算倍率
for _, bt := range params.BetTypes {
    bt.Multiplier = (float64(params.TotalNumbers) / float64(bt.Coverage)) * (rtp / 100.0)
}
```

### 关键参数
- **RTP**: 96%（配置可调）
- **总数字数**: 37（欧式轮盘）
- **倍率公式**: `(37 / coverage) × 0.96`
- **金额精度**: 8 位小数

### 赔率表

| 投注类型 | 覆盖数 | 命中概率 | 倍率 |
|---------|--------|---------|------|
| straight | 1 | 1/37 (2.70%) | 35.52x |
| split | 2 | 2/37 (5.41%) | 17.76x |
| street | 3 | 3/37 (8.11%) | 11.84x |
| corner | 4 | 4/37 (10.81%) | 8.88x |
| sixline | 6 | 6/37 (16.22%) | 5.92x |
| column | 12 | 12/37 (32.43%) | 2.96x |
| dozen | 12 | 12/37 (32.43%) | 2.96x |
| red | 18 | 18/37 (48.65%) | 1.97x |
| black | 18 | 18/37 (48.65%) | 1.97x |
| odd | 18 | 18/37 (48.65%) | 1.97x |
| even | 18 | 18/37 (48.65%) | 1.97x |
| low | 18 | 18/37 (48.65%) | 1.97x |
| high | 18 | 18/37 (48.65%) | 1.97x |

### 多投注结算

单次请求可包含多个投注，分别独立结算：

```
totalPayout = Σ (命中投注的 amount × multiplier)
winAmount = totalPayout - totalBetAmount
multiplier = totalPayout / totalBetAmount（总倍率）
isWin = len(winningBets) > 0
```

## 可证明公平实现

### 随机数生成算法

```go
func GenerateRouletteNumber(serverSeed, clientSeed string, nonce int64, totalNumbers int) int {
    seedStr := random.BuildSeed(serverSeed, clientSeed, nonce)
    return int(random.Int64n(seedStr, int64(totalNumbers)))
}
```

### 生成过程

```
1. 组合种子：seedStr = "serverSeed:clientSeed:nonce"
2. 计算哈希：hash = SHA256(seedStr)
3. 取前 8 位十六进制：hashFirst8 = hash[:8]
4. 转换为整数：hashValue = parseInt(hashFirst8, 16)
5. 计算中奖号码：winningNumber = hashValue % 37  → 0-36
```

### 颜色判定

```go
func GetNumberColor(number int, redNumbers []int) string {
    if number == 0 {
        return "green"
    }
    if contains(redNumbers, number) {
        return "red"
    }
    return "black"
}
```

### 公平性保障
- **种子顺序**: `serverSeed:clientSeed:nonce`（无 index，即时游戏只需一个随机数）
- **客户端种子**: 玩家提供，确保随机性不受服务器控制
- **服务器种子**: 游戏开始前生成，结束后公开供验证
- **Nonce**: 递增计数器，确保每局唯一性

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/roulette/service.go
type Service struct {
    *games.InstantGameBase[*roulettegame.Game, *v1.RoulettePlaceBetRequest]
}
```

### 2. 核心数据结构
```go
type Game struct {
    game.InstantGame
    Bets   []Bet   `json:"-"`
    Params *Params `json:"-"`

    WinningNumber int         `json:"winningNumber"`
    Color         string      `json:"color"`
    WinningBets   []BetResult `json:"winningBets"`
}

type Params struct {
    TotalNumbers int
    RedNumbers   []int
    BlackNumbers []int
    Multipliers  map[string]float64
    RTP          float64
}

type Bet struct {
    Type    string
    Numbers []int
    Amount  decimal.Decimal
}

type BetResult struct {
    BetType    string          `json:"betType"`
    Numbers    []int           `json:"numbers"`
    Stake      decimal.Decimal `json:"stake"`
    Multiplier float64         `json:"multiplier"`
    Payout     decimal.Decimal `json:"payout"`
}
```

`game.InstantGame` 内嵌 `game.Game`，包含 `RoundID`、`BetAmount`（decimal.Decimal）、`ClientSeed`、`ServerSeed`、`Nonce`、`FinalPayout`（decimal.Decimal）、`RTP`、`IsWin`、`IsDemo` 等基础字段。

### 3. WebSocket 接口

#### 游戏控制
- **placeBet**: 下注并立即出结果（单一 action）

### 4. 投注验证

Service 层对每种投注类型进行参数验证：

| 投注类型 | numbers 数量 | 取值范围 |
|---------|-------------|---------|
| straight | 1 | 0-36 |
| split | 2 | - |
| street | 3 | - |
| corner | 4 | - |
| sixline | 6 | - |
| column | 1 | 1-3 |
| dozen | 1 | 1-3 |
| red/black/odd/even/low/high | 不需要 | - |

### 5. 输赢判定逻辑

```go
func (g *Game) isWinningBet(bet Bet, winningNumber int) bool {
    switch bet.Type {
    case straight, split, street, corner, sixline:
        return contains(bet.Numbers, winningNumber)
    case column:
        return winningNumber > 0 && winningNumber%3 == column%3
    case dozen:
        // dozen=1: 1-12, dozen=2: 13-24, dozen=3: 25-36
    case red:
        return contains(redNumbers, winningNumber)
    case black:
        return contains(blackNumbers, winningNumber)
    case odd:
        return winningNumber > 0 && winningNumber%2 == 1
    case even:
        return winningNumber > 0 && winningNumber%2 == 0
    case low:
        return winningNumber >= 1 && winningNumber <= 18
    case high:
        return winningNumber >= 19 && winningNumber <= 36
    }
}
```

## 与 Dragon Tiger 的对比

| 特性 | Roulette | Dragon Tiger |
|------|----------|-------------|
| 游戏类型 | Instant 游戏 | Instant 游戏 |
| 赔率类型 | formula | formula |
| 投注方式 | 多投注类型，单次可多注 | 三区域投注（龙/虎/和） |
| 随机生成 | 单个数字（0-36） | 两张牌（各 1-13） |
| 种子格式 | `serverSeed:clientSeed:nonce` | `serverSeed:clientSeed:nonce:index` |
| RTP | 96% | 97% |
| 投注类型数 | 13 种 | 3 种 |
| 倍率计算 | `(totalNumbers / coverage) × RTP` | 固定赔率 × RTP |
