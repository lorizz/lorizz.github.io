# Tutorial — Step 5: Server Bootstrap

The server Bootstrap is a `Script` in `ServerScriptService/Server/`. It is the composition root — the only place where everything is constructed and wired together.

---

## Bootstrap.server.lua

```lua
-- Server/Bootstrap.server.lua
--!nostrict
local Players          = game:GetService("Players")
local ServiceLocator   = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)
local NetBridge        = require(game.ReplicatedStorage.Shared.Framework.NetBridge)
local PlayerRepository = require(script.Parent.Repositories.PlayerRepository)
local DataService      = require(script.Parent.Services.DataService)

-- 1. Construct (Repository first, then Services that depend on it)
local repo    = PlayerRepository.new()
local dataSvc = DataService.new(repo)

-- 2. Register Services in ServiceLocator
ServiceLocator.Register("DataService", dataSvc)

-- 3. Create network events before any client connects
NetBridge.CreateEvent("CoinsUpdate")

-- 4. Player lifecycle
local function onPlayerAdded(player)
    dataSvc:LoadPlayer(player)            -- yields: waits for DataStore
    local data = dataSvc:GetData(player)
    if not data then return end
    NetBridge.FireClient("CoinsUpdate", player, data.coins)
end

Players.PlayerAdded:Connect(onPlayerAdded)

-- Handle players already in-game when this script initializes
for _, player in Players:GetPlayers() do
    task.spawn(onPlayerAdded, player)
end

Players.PlayerRemoving:Connect(function(player)
    dataSvc:UnloadPlayer(player)
end)

-- Guarantees saves on server shutdown (Stop in Studio, normal shutdown)
game:BindToClose(function()
    for _, player in Players:GetPlayers() do
        dataSvc:UnloadPlayer(player)
    end
end)

-- 5. Wire Service Signals to NetBridge
-- The Service fires a Signal. Bootstrap pushes it to the client.
dataSvc.OnCoinsChanged:Connect(function(player, newCoins)
    NetBridge.FireClient("CoinsUpdate", player, newCoins)
end)

-- 6. Game loop: +1 coin per second for every player
task.spawn(function()
    while true do
        task.wait(1)
        for _, player in Players:GetPlayers() do
            dataSvc:AddCoins(player, 1)
        end
    end
end)
```

---

## Why `task.spawn` in the player loop

`Players.PlayerAdded` fires in its own thread per player. However, `LoadPlayer` yields (DataStore call), which means if we called it directly without spawning, the first player's load would block until it completed before any other `PlayerAdded` logic ran for the same player.

More importantly, the `for _, player in Players:GetPlayers()` catch-up loop at the bottom needs `task.spawn` explicitly because it runs synchronously — without it, the loop would block on the first player's DataStore call before moving to the second player.

```lua
-- ✅ Each player loads in its own thread
for _, player in Players:GetPlayers() do
    task.spawn(onPlayerAdded, player)
end

-- ❌ Blocks: player 2 waits for player 1's DataStore call to finish
for _, player in Players:GetPlayers() do
    onPlayerAdded(player)
end
```

---

## Why `BindToClose`

When you press Stop in Studio, or when a live server shuts down, `PlayerRemoving` may not complete before the process is killed. `BindToClose` registers a callback that Roblox will wait up to 30 seconds for before shutting down. This guarantees that every player's `UnloadPlayer` — and therefore their `Save` — runs to completion.

Without it, players lose data every time the server restarts.

---

Next: [Step 6 — Cubit](06-cubit.md)
