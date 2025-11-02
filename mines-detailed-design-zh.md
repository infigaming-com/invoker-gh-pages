# Mines（扫雷）游戏详细设计文档 ✅ 已实现

## 游戏概述

Mines 是一个策略性扫雷游戏，玩家需要在网格中避开地雷，每成功揭示一个安全格子都会增加赔率倍数。

## 游戏规则

### 基本玩法
1. **选择配置**：
   - 网格尺寸：3×3（9格）、5×5（25格）、7×7（49格）
   - 地雷数量：1-24个（取决于网格大小）
   - 默认配置：5×5网格

2. **游戏流程**：
   - 玩家逐个点击格子揭示内容
   - 揭示安全格子获得倍数增长
   - 点到地雷游戏结束，失去投注
   - 可随时兑现当前倍数奖励
   - 揭示所有安全格子获得最大奖励

3. **游戏状态**：
   - `in_progress` - 游戏进行中
   - `won` - 获胜（揭示所有安全格子）
   - `lost` - 失败（点到地雷）
   - `cashed_out` - 主动提现

### 网格配置

| 网格尺寸 | 总格子数 | 最大地雷数 | 最小地雷数 |
|----------|----------|------------|------------|
| 3×3      | 9        | 8          | 1          |
| 5×5      | 25       | 24         | 1          |
| 7×7      | 49       | 48         | 1          |

## 赔率计算公式

### 核心算法
```go
// 当前倍数计算
multiplier := 1.0
for i := 0; i < safeRevealed; i++ {
    remainingSafe := float64(safeTiles - i)
    remainingTotal := float64(totalTiles - i)
    probability := remainingSafe / remainingTotal
    multiplier *= (1 / probability)
}
multiplier *= (1 - HouseEdge) // HouseEdge = 0.01 (1%)
```

### 关键参数
- **House Edge（庄家优势）**：1%
- **安全格子数**：总格子数 - 地雷数
- **概率计算**：剩余安全格子 ÷ 剩余总格子
- **累积倍数**：每步倍数相乘

### 赔率示例（5×5网格）

| 地雷数 | 第1步倍数 | 第5步倍数 | 第10步倍数 | 最大倍数 |
|--------|-----------|-----------|------------|----------|
| 1      | 1.04×     | 1.24×     | 2.09×      | 24.75×   |
| 3      | 1.13×     | 1.86×     | 5.26×      | 120×     |
| 5      | 1.24×     | 2.79×     | 14.68×     | 840×     |
| 10     | 1.65×     | 8.25×     | 231×       | 5000×    |

## 可证明公平实现

### 地雷位置生成算法
```go
func generateMinePositions(clientSeed, serverSeed string, nonce int64, minesCount int) []int {
    positions := make([]int, 0, minesCount)
    used := make(map[int]bool)
    
    for i := 0; i < minesCount; i++ {
        // 生成哈希
        seedStr := fmt.Sprintf("%s:%s:%d:%d", clientSeed, serverSeed, nonce, i)
        hash := sha256.Sum256([]byte(seedStr))
        hashHex := hex.EncodeToString(hash[:])
        
        // 转换为位置
        hashValue := new(big.Int)
        hashValue.SetString(hashHex[:8], 16)
        position := hashValue.Mod(hashValue, big.NewInt(int64(gridSize))).Int64()
        
        // 处理碰撞（线性探测）
        for used[int(position)] {
            position = (position + 1) % int64(gridSize)
        }
        
        used[int(position)] = true
        positions = append(positions, int(position))
    }
    
    return positions
}
```

### 公平性保障
- **客户端种子**：玩家提供，确保随机性不受服务器控制
- **服务器种子**：游戏开始前生成，结束后公开供验证
- **Nonce**：递增计数器，确保每局唯一性
- **SHA256哈希**：加密哈希函数保证不可预测性
- **验证接口**：提供独立验证函数供玩家验证

### 验证 API 接口

系统提供 RESTful API 用于独立验证游戏结果：

**端点**: `POST /v1/fairness/mines/verify`

**特点**：
- 需要 JWT Token 认证
- 使用与游戏相同的算法
- 支持所有网格类型和地雷配置
- 返回确定性结果供对比验证

**参数验证规则**：
```go
func validateMinesRequest(req *pb.VerifyMinesResultRequest) error {
    // 客户端种子和服务器种子不能为空
    if req.ClientSeed == "" || req.ServerSeed == "" {
        return errors.BadRequest("INVALID_SEED", "种子不能为空")
    }

    // Nonce 必须 >= 0
    if req.Nonce < 0 {
        return errors.BadRequest("INVALID_NONCE", "Nonce值必须大于等于0")
    }

    // 网格类型必须有效
    gridConfig, exists := GridConfigs[req.GridType]
    if !exists {
        return errors.BadRequest("INVALID_GRID_TYPE", "无效的网格类型")
    }

    // 地雷数量必须在有效范围内
    if req.MinesCount <= 0 || req.MinesCount >= gridConfig.MaxMines {
        return errors.BadRequest("INVALID_MINES_COUNT", "地雷数量无效")
    }

    return nil
}
```

**使用场景**：
- 玩家验证游戏结果公平性
- 第三方审计机构独立验证
- 开发者测试和调试
- 前端实现客户端验证功能

**与 WebSocket 接口的关系**：
- WebSocket 接口：游戏进行时使用，结果中包含哈希后的服务器种子
- 验证 API：游戏结束后使用，需要揭示的服务器种子
- 两者使用相同的 `CalculateMinePositions()` 函数确保一致性

## 系统实现细节

### 1. 服务架构
```go
// internal/service/games/mines/handler.go
type GameService struct {
    base            *base.SessionGameService
    serverSeedRepo  data.ServerSeedRepo
    gameSessionRepo data.GameSessionRepo
    gameResultRepo  data.GameResultRepo
    userRepo        data.UserRepo
    aggregatorRepo  data.AggregatorRepo
    clientManager   *aggregator.ClientManager
    idGenerator     *biz.IDGeneratorUsecase
    
    // 会话管理
    activeGamesByUser  map[int64]map[string]*GameInstance
    activeGamesByRound map[string]*GameInstance
}
```

### 2. 核心数据结构
```go
// 游戏主体
type MinesGame struct {
    gridConfig      GridConfig  // 网格配置
    betAmount       float64     // 投注金额
    minesCount      int         // 地雷数量
    clientSeed      string      // 客户端种子
    serverSeed      string      // 服务器种子
    nonce           int64       // 随机数
    
    roundID         string      // 回合ID
    status          GameStatus  // 游戏状态
    minePositions   []int       // 地雷位置
    revealedTiles   []int       // 已揭示格子
    safeRevealed    int         // 已揭示安全格子数
    
    finalMultiplier float64     // 最终倍数
    payout          float64     // 奖金
}

// 网格配置
type GridConfig struct {
    Rows      int  // 行数
    Cols      int  // 列数
    TotalSize int  // 总格子数
    MaxMines  int  // 最大地雷数
}
```

### 3. WebSocket接口
- **PLACE_BET**：开始新游戏
  - 参数：`minesCount`、`gridType`
- **MINES_REVEAL_TILE**：揭示格子
  - 参数：`position`（0-based索引）
- **MINES_CASH_OUT**：主动提现
- **MINES_CHECK_ACTIVE**：检查活跃游戏
- **MINES_RESUME_GAME**：恢复游戏

### 4. 会话管理

#### 生命周期
1. **创建**：PlaceBet时创建新会话
2. **持久化**：状态保存到数据库
3. **恢复**：支持断线重连
4. **清理**：定时清理非活跃会话

#### 状态序列化
```go
gameData := map[string]interface{}{
    "mines_count":    minesCount,
    "grid_type":      gridType,
    "mine_positions": minePositions,
    "revealed_tiles": revealedTiles,
    "client_seed":    clientSeed,
}
```

## 前端实现建议

### 1. 界面设计
- **网格显示**：
  - 使用方格按钮表示格子
  - 已揭示安全格子显示绿色
  - 地雷爆炸显示红色
  - 未揭示格子显示灰色

- **信息面板**：
  - 当前倍数（实时更新）
  - 潜在赢利金额
  - 剩余安全格子数
  - 下一步倍数预测

- **倍率显示规范**：
  - **重要**：倍率显示应使用向下取整（floor）而非四舍五入（round）
  - 示例：1.997368 应显示为 1.99x，而非 2.00x
  - 原因：避免给玩家造成误导，显示的倍率应始终小于或等于实际倍率
  - 实现建议：
    ```javascript
    // JavaScript 示例
    const displayMultiplier = Math.floor(actualMultiplier * 100) / 100;

    // 或者使用 toFixed + parseFloat
    const displayMultiplier = parseFloat(actualMultiplier.toFixed(2));
    if (displayMultiplier > actualMultiplier) {
        displayMultiplier = Math.floor(actualMultiplier * 100) / 100;
    }
    ```

### 2. 交互设计
- **开始游戏**：
  - 选择网格大小（下拉菜单）
  - 选择地雷数量（滑块或输入）
  - 设置投注金额
  - 输入客户端种子

- **游戏中**：
  - 点击格子揭示
  - 兑现按钮（显示当前奖金）
  - 自动选择模式（可选）

### 3. 动画效果
- 揭示动画：翻转效果
- 爆炸动画：地雷爆炸特效
- 倍数增长：数字滚动动画
- 获胜庆祝：全屏特效

### 4. 历史记录
- 显示地雷位置图
- 玩家选择路径
- 最终倍数和奖金
- 可证明公平验证链接

## 性能优化

1. **预计算优化**：
   - 启动时预计算常用配置的概率表
   - 缓存倍数计算结果

2. **并发处理**：
   - 使用读写锁保护共享状态
   - 支持多用户并发游戏

3. **内存管理**：
   - 定期清理过期会话
   - 使用对象池复用游戏实例