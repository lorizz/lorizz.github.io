# Bootstrap

The two Bootstrap files are the **composition roots** of the application. They are the only place where instances are constructed and wired together. Nothing outside of Bootstrap calls `.new()` on a Service, Repository, or Cubit.

There are exactly two Bootstrap files:
- `Server/Bootstrap.server.lua` — a `Script` in `ServerScriptService`
- `Client/Bootstrap.client.lua` — a `LocalScript` in `StarterPlayerScripts`

---

## Server Bootstrap

Responsibilities in order:

1. Construct Repositories
2. Construct Services (inject Repositories)
3. Register Services in ServiceLocator
4. Create NetBridge events and functions
5. Call `Controller.Init()` for each Controller
6. Connect player lifecycle (PlayerAdded, PlayerRemoving, BindToClose)
7. Connect Service Signals to NetBridge pushes
8. Start any server-side loops

```lua
--!nostrict
local Players          = game:GetService("Players")
local ServiceLocator   = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)
local NetBridge        = require(game.ReplicatedStorage.Shared.Framework.NetBridge)

-- Repositories
local PlayerRepository = require(script.Parent.Repositories.PlayerRepository)

-- Services
local DataService      = require(script.Parent.Services.DataService)
local InventoryService = require(script.Parent.Services.InventoryService)

-- Controllers
local PlayerController = require(script.Parent.Controllers.PlayerController)

-- 1. Construct (dependency order: low-level first)
local repo    = PlayerRepository.new()
local dataSvc = DataService.new(repo)
local invSvc  = InventoryService.new(dataSvc)

-- 2. Register
ServiceLocator.Register("DataService", dataSvc)
ServiceLocator.Register("InventoryService", invSvc)

-- 3. Network
NetBridge.CreateEvent("CoinsUpdate")
NetBridge.CreateEvent("InventoryUpdate")

-- 4. Controllers
PlayerController.Init()

-- 5. Player lifecycle
local function onPlayerAdded(player)
    dataSvc:LoadPlayer(player)
    local data = dataSvc:GetData(player)
    if not data then return end
    NetBridge.FireClient("CoinsUpdate", player, data.coins)
    NetBridge.FireClient("InventoryUpdate", player, data.inventory)
end

Players.PlayerAdded:Connect(onPlayerAdded)

-- Handle players already in-game when the script initializes
for _, player in Players:GetPlayers() do
    task.spawn(onPlayerAdded, player)
end

Players.PlayerRemoving:Connect(function(player)
    dataSvc:UnloadPlayer(player)
end)

-- Guarantee saves on server shutdown
game:BindToClose(function()
    for _, player in Players:GetPlayers() do
        dataSvc:UnloadPlayer(player)
    end
end)

-- 6. Wire Signals to NetBridge
dataSvc.OnCoinsChanged:Connect(function(player, newCoins)
    NetBridge.FireClient("CoinsUpdate", player, newCoins)
end)

invSvc.OnInventoryChanged:Connect(function(player, inventory)
    NetBridge.FireClient("InventoryUpdate", player, inventory)
end)

-- 7. Game loops
task.spawn(function()
    while true do
        task.wait(1)
        for _, player in Players:GetPlayers() do
            dataSvc:AddCoins(player, 1)
        end
    end
end)
```

### Why `BindToClose`?

`PlayerRemoving` fires when a player leaves, but when the server shuts down (or you press Stop in Studio), `PlayerRemoving` may not complete before the process is killed. `BindToClose` gives the server up to 30 seconds to finish before shutdown, guaranteeing that `UnloadPlayer` — and therefore `Save` — completes.

```lua
-- Always include this. Without it, data is lost on server shutdown.
game:BindToClose(function()
    for _, player in Players:GetPlayers() do
        dataSvc:UnloadPlayer(player)
    end
end)
```

---

## Client Bootstrap

Responsibilities in order:

1. Resolve the player and GUI
2. Construct Cubits with initial state
3. Register Cubits in ServiceLocator
4. Mount Views (pass Cubits explicitly)
5. Connect NetBridge listeners to Cubit methods

```lua
--!nostrict
local ServiceLocator     = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)
local NetBridge          = require(game.ReplicatedStorage.Shared.Framework.NetBridge)
local Context            = require(game.ReplicatedStorage.Shared.Framework.Context)

-- Cubits
local HUDCubit           = require(script.Parent.Cubits.HUDCubit)
local InventoryCubit     = require(script.Parent.Cubits.InventoryCubit)

-- Views
local HUDView            = require(script.Parent.Views.HUDView)
local InventoryView      = require(script.Parent.Views.InventoryView)

local player = Context.GetPlayer()
local gui    = player.PlayerGui:WaitForChild("MainGui")

-- 1. Construct Cubits
local hudCubit = HUDCubit.new(0)
local invCubit = InventoryCubit.new({})

-- 2. Register (register before constructing Cubits that depend on others)
ServiceLocator.Register("HUDCubit", hudCubit)
ServiceLocator.Register("InventoryCubit", invCubit)

-- 3. Mount Views — Cubits passed explicitly, not resolved inside Views
HUDView.Mount(gui, hudCubit)
InventoryView.Mount(gui, invCubit)

-- 4. Server → Cubit pipeline
NetBridge.OnClient("CoinsUpdate", function(newCoins)
    hudCubit:SetCoins(newCoins)
end)

NetBridge.OnClient("InventoryUpdate", function(inventory)
    invCubit:SetInventory(inventory)
end)
```

---

## Registration order matters

If a Cubit depends on another Cubit via ServiceLocator, the dependency must be registered first.

```lua
-- ✅ Correct: InventoryCubit registered before NotificationCubit is constructed
local invCubit   = InventoryCubit.new({})
ServiceLocator.Register("InventoryCubit", invCubit)

local notifCubit = NotificationCubit.new()   -- resolves InventoryCubit internally
ServiceLocator.Register("NotificationCubit", notifCubit)

-- ❌ Wrong: NotificationCubit constructed before InventoryCubit is registered
local notifCubit = NotificationCubit.new()   -- ServiceLocator.Get("InventoryCubit") throws
local invCubit   = InventoryCubit.new({})
```

The same rule applies on the server — Repositories before Services, Services before Controllers.

---

## Quick reference

| What | Where it lives | Who constructs it |
|---|---|---|
| Repository | `Bootstrap.server.lua` | Bootstrap directly |
| Service | `Bootstrap.server.lua` | Bootstrap directly |
| Controller | `Bootstrap.server.lua` | `Controller.Init()` |
| Cubit | `Bootstrap.client.lua` | Bootstrap directly |
| View | `Bootstrap.client.lua` | `View.Mount(gui, cubit)` |
