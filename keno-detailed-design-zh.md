# Keno（基诺）游戏详细设计文档 ✅ 已实现

## 配置文件

### 游戏配置 (`configs/games/keno.json`)
```json
{
  "gameId": "inhousegame:keno",
  "status": "active",
  "oddsType": "table",
  "gameParameters": {
    "defaultDifficulty": "classic",
    "minSpots": 1,
    "maxSpots": 10
  },
  "payoutTables": {
    "low": {
      "spots": {
        "1": {"entries": [{"hits": 0, "payout": 0.7}, {"hits": 1, "payout": 1.85}]},
        "5": {"entries": [{"hits": 0, "payout": 0}, {"hits": 1, "payout": 0}, {"hits": 2, "payout": 1.5}, {"hits": 3, "payout": 4.2}, {"hits": 4, "payout": 13}, {"hits": 5, "payout": 300}]}
      }
    },
    "classic": { ... },
    "medium": { ... },
    "high": { ... }
  }
}
```

### 配置说明
- **gameId**: 游戏唯一标识符，格式为 `"inhousegame:keno"`
- **status**: 游戏状态（`active` 启用，`disabled` 禁用）
- **oddsType**: 赔率类型，`table` 表示表驱动（赔率由 payoutTables 决定）
- **gameParameters**: 游戏参数
  - **defaultDifficulty**: 默认难度（`classic`）
  - **minSpots**: 最少选号数量（1）
  - **maxSpots**: 最多选号数量（10）
- **payoutTables**: 赔率表配置，支持 4 种难度模式
  - `low`: 低风险模式（高频小奖）
  - `classic`: 经典模式（平衡模式，默认）
  - `medium`: 中等风险模式
  - `high`: 高风险模式（高风险高回报）
  - 每个难度包含 1-10 个选号数量（spots）的赔率表
  - 赔率表格式：`{ "spots": { "N": { "entries": [{"hits": H, "payout": P}, ...] } } }`

### 配置加载
- **加载时机**: 服务器启动时，GameRegistry 自动从 `configs/games/` 目录加载所有 `.json` 配置文件
- **使用方式**: 游戏服务通过 `gameRegistry.GetGame("inhousegame:keno")` 获取配置，加载赔率表用于游戏结算
- **配置验证**: Service 层和 Biz 层都会验证配置完整性，配置缺失时返回错误
- **投注限额**: 从 `configs/currencies.json` 自动生成 BetInfo

### 赔率表结构示例
```json
"classic": {
  "spots": {
    "5": {
      "entries": [
        {"hits": 0, "payout": 0},      // 匹配0个数字，赔率0
        {"hits": 1, "payout": 0.25},   // 匹配1个数字，赔率0.25×
        {"hits": 2, "payout": 1.4},    // 匹配2个数字，赔率1.4×
        {"hits": 3, "payout": 4.1},    // 匹配3个数字，赔率4.1×
        {"hits": 4, "payout": 16.5},   // 匹配4个数字，赔率16.5×
        {"hits": 5, "payout": 36}      // 匹配5个数字（全中），赔率36×
      ]
    }
  }
}
```

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

## 可证明公平性验证

Keno 游戏实现了完整的可证明公平（Provably Fair）机制，玩家可以独立验证每一局游戏的公平性。

### 验证接口

**接口地址**：`POST /v1/fairness/keno/verify`

**认证方式**：需要 JWT Token

**请求参数**：

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `clientSeed` | string | 是 | 客户端种子（8-256字符） |
| `serverSeed` | string | 是 | 服务端种子（已揭示） |
| `nonce` | int64 | 是 | nonce 值（≥0） |
| `selectedNumbers` | int32[] | 是 | 玩家选择的数字（1-10个，范围1-40） |
| `difficulty` | string | 是 | 难度模式（low/classic/medium/high） |

**响应结果**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `drawnNumbers` | int32[] | 验证计算出的系统开出的10个号码 |
| `matchedNumbers` | int32[] | 验证计算出的匹配数字列表 |
| `matchCount` | int32 | 验证计算出的匹配数量 |
| `multiplier` | double | 验证计算出的赔率倍数 |

### 验证原理

#### 1. 种子组合

验证使用与游戏相同的种子组合方式：

```
seedString = "serverSeed:clientSeed:nonce"
```

**种子顺序说明**：
- 统一规范：所有游戏都使用 `"serverSeed:clientSeed:nonce"` 顺序
- 确保一致性：验证时必须使用相同的顺序

#### 2. 哈希计算

使用 SHA256 对种子字符串进行哈希：

```go
hash := sha256.Sum256([]byte(seedStr))
```

#### 3. Fisher-Yates 洗牌算法

使用哈希值作为随机源，执行 Fisher-Yates 洗牌算法从 1-40 中抽取 10 个不重复数字：

```go
func DrawKenoNumbers(clientSeed, serverSeed string, nonce int64) []int {
    // 生成种子字符串
    seedStr := fmt.Sprintf("%s:%s:%d", serverSeed, clientSeed, nonce)
    hash := sha256.Sum256([]byte(seedStr))

    // 初始化数字池（1-40）
    numbers := make([]int, 40)
    for i := 0; i < 40; i++ {
        numbers[i] = i + 1
    }

    // Fisher-Yates 洗牌抽取前10个
    for i := 0; i < 10; i++ {
        hashOffset := i * 4
        if hashOffset+4 > len(hash) {
            // 哈希不够用时重新生成
            seedStr = fmt.Sprintf("%s:%d", hex.EncodeToString(hash[:]), i)
            hash = sha256.Sum256([]byte(seedStr))
            hashOffset = 0
        }

        randomBytes := hash[hashOffset : hashOffset+4]
        randomIndex := binary.BigEndian.Uint32(randomBytes) % uint32(40-i)

        // 交换
        numbers[i], numbers[i+int(randomIndex)] = numbers[i+int(randomIndex)], numbers[i]
    }

    // 返回前10个数字并排序
    result := numbers[:10]
    sort.Ints(result)
    return result
}
```

#### 4. 匹配计算

计算玩家选择的数字与系统开出的数字的匹配情况：

```go
func CalculateMatches(selected []int32, drawn []int) []int32 {
    drawnMap := make(map[int]bool)
    for _, num := range drawn {
        drawnMap[num] = true
    }

    matched := []int32{}
    for _, num := range selected {
        if drawnMap[int(num)] {
            matched = append(matched, num)
        }
    }
    return matched
}
```

#### 5. 赔率查询

根据选择数量、匹配数量和难度模式查询赔率表：

```go
func GetMultiplier(spots, matches int, difficulty string, rtp float64) float64 {
    // 从全局赔率表中查询基础赔率
    basePayout := GlobalPayoutTable.Difficulties[difficulty].Payouts[spots][matches]

    // 应用 RTP 调整
    return basePayout * (rtp / 100.0)
}
```

### 验证步骤

#### 步骤 1: 获取种子信息

从游戏历史记录中获取：
- **clientSeed**: 游戏前设置的客户端种子
- **serverSeed**: 游戏结束后揭示的服务端种子（完整种子，非哈希）
- **nonce**: 该局游戏的 nonce 值
- **selectedNumbers**: 玩家选择的数字
- **difficulty**: 选择的难度模式

#### 步骤 2: 调用验证接口

使用获取的参数调用验证 API。

#### 步骤 3: 对比结果

将验证返回的结果与游戏实际结果进行对比：
- `drawnNumbers`: 系统开出的 10 个数字是否完全一致（包括顺序）
- `matchedNumbers`: 匹配的数字列表是否一致
- `matchCount`: 匹配数量是否一致
- `multiplier`: 赔率倍数是否一致

#### 步骤 4: 验证服务端种子承诺

验证服务端在游戏前提供的种子哈希是否与揭示的种子一致：

```go
revealedSeedHash := sha256.Sum256([]byte(revealedServerSeed))
if hex.EncodeToString(revealedSeedHash[:]) == committedSeedHash {
    // 种子承诺有效，服务端未作弊
}
```

### 使用场景

1. **玩家自主验证**
   - 玩家可以随时验证历史游戏记录
   - 确认游戏结果的真实性和公平性

2. **第三方审计**
   - 独立审计机构可以批量验证游戏记录
   - 验证平台的公平性和合规性

3. **争议解决**
   - 当出现争议时，提供可验证的证据
   - 通过数学证明解决纠纷

### 安全保证

1. **确定性算法**
   - 相同的种子和 nonce 必然产生相同的结果
   - 结果完全可复现和验证

2. **种子承诺机制**
   - 游戏前只公开服务端种子的哈希（承诺）
   - 游戏后揭示完整种子，确保服务端无法作弊

3. **客户端参与**
   - 客户端种子由玩家设置，服务端无法预测
   - 双向参与确保随机性

4. **Nonce 机制**
   - 每次游戏 nonce 递增
   - 防止相同种子对产生重复结果

### 技术细节

#### 哈希不足处理

当需要的随机字节超过单次 SHA256 输出（32字节）时：

```go
if hashOffset+4 > len(hash) {
    // 使用当前哈希作为新的种子继续生成
    seedStr = fmt.Sprintf("%s:%d", hex.EncodeToString(hash[:]), i)
    hash = sha256.Sum256([]byte(seedStr))
    hashOffset = 0
}
```

#### 模运算偏差处理

使用 32 位随机数确保足够的随机性，模运算产生的偏差在可接受范围内：

```go
randomIndex := binary.BigEndian.Uint32(randomBytes) % uint32(40-i)
```

对于 1-40 的范围，偏差影响微乎其微（< 0.00001%）。

### 验证工具建议

建议前端实现验证工具，让玩家可以：
1. 输入游戏参数（种子、nonce、选择的数字、难度）
2. 本地计算哈希和结果（使用 WebCrypto API）
3. 与服务端验证接口结果对比
4. 显示详细的验证过程和结果

**示例验证工具界面**：
- 种子输入区（clientSeed、serverSeed、nonce）
- 游戏参数输入（selectedNumbers、difficulty）
- "验证"按钮
- 结果对比显示（本地计算 vs 服务端返回）
- 验证通过/失败状态指示