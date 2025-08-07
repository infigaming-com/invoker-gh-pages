# WebSocket 消息路由修复说明

## 问题描述
Dice 和 Mines 游戏适配器都注册了相同的 "PLACE_BET" 消息类型，导致后注册的 Mines 适配器覆盖了 Dice 的处理器，使得 Dice 游戏无法正常工作。

## 修复方案
基于用户登录时的 GameID（`inhousegame:dice` 或 `inhousegame:mines`）进行智能路由。

### 实施步骤

1. **修改 DefaultMessageHandler 结构**
   - 添加 `diceAdapter` 和 `minesAdapter` 字段
   - 添加 `SetDiceAdapter` 和 `SetMinesAdapter` 方法

2. **实现统一的 PLACE_BET 处理器**
   - 在 `DefaultMessageHandler` 中添加 `handlePlaceBet` 方法
   - 根据 `user.GameID` 路由到正确的游戏适配器
   - 使用 `pkg/consts` 中定义的常量进行比较

3. **修改适配器注册方式**
   - Dice 和 Mines 适配器不再直接注册 "PLACE_BET"
   - 改为通过 `SetDiceAdapter`/`SetMinesAdapter` 设置处理函数

### 代码更改

#### handler.go
```go
// 添加游戏适配器字段
type DefaultMessageHandler struct {
    // ... 其他字段
    diceAdapter  MessageHandlerFunc
    minesAdapter MessageHandlerFunc
}

// 添加设置方法
func (h *DefaultMessageHandler) SetDiceAdapter(adapter MessageHandlerFunc) {
    h.diceAdapter = adapter
}

func (h *DefaultMessageHandler) SetMinesAdapter(adapter MessageHandlerFunc) {
    h.minesAdapter = adapter
}

// 统一的 PLACE_BET 处理器
func (h *DefaultMessageHandler) handlePlaceBet(ctx context.Context, conn *Connection, msg *v1.WebSocketMessage) error {
    user := conn.GetUser()
    switch user.GameID {
    case consts.GameIDDice:
        return h.diceAdapter(ctx, conn, msg)
    case consts.GameIDMines:
        return h.minesAdapter(ctx, conn, msg)
    default:
        return h.sendErrorResponse(conn, msg.Id, "UNSUPPORTED_GAME", fmt.Sprintf("Game %s is not supported", user.GameID))
    }
}
```

#### dice_adapter.go 和 mines_adapter.go
```go
// 修改前
messageHandler.RegisterHandler("PLACE_BET", adapter.handlePlaceBet)

// 修改后
messageHandler.SetDiceAdapter(adapter.handlePlaceBet)  // 或 SetMinesAdapter
```

## 测试方法

1. **启动服务器**
   ```bash
   make run
   ```

2. **测试 Dice 游戏**
   - 使用 JWT token 中 `game_id: "inhousegame:dice"` 登录
   - 发送 PLACE_BET 消息，应该路由到 Dice 适配器

3. **测试 Mines 游戏**
   - 使用 JWT token 中 `game_id: "inhousegame:mines"` 登录
   - 发送 PLACE_BET 消息，应该路由到 Mines 适配器

4. **查看日志**
   - 应该看到类似 "Routing PLACE_BET to Dice adapter" 或 "Routing PLACE_BET to Mines adapter" 的日志

## 优势

1. **符合现有架构**：用户在登录时通过 JWT 指定游戏，避免混淆
2. **性能更好**：不需要每次解析 gameParams
3. **更安全**：防止用户发送错误的游戏参数
4. **易于扩展**：添加新游戏只需在 switch 语句中添加新的 case
5. **使用官方常量**：避免硬编码，提高代码可维护性