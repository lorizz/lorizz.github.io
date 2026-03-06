# Tutorial — Step 4: Service

`DataService` is the single source of truth for player data on the server. It holds an in-memory cache, exposes mutation methods, and fires a Signal when coins change. It never calls DataStore directly — it delegates to the Repository.

---

## DataService

```lua
-- Server/Services/DataService.lua
--!nostrict

--- @class DataService
--- Single source of truth for all player data on the server.

local Signal      = require(game.ReplicatedStorage.Shared.Framework.Signal)
local PlayerTypes = require(game.ReplicatedStorage.Shared.Types.PlayerTypes)

local DataService = {}
DataService.__index = DataService

--- @param repo PlayerRepository
--- @return DataService
function DataService.new(repo)
    return setmetatable({
        _repo  = repo,
        _cache = {},
        --- Fired when a player's coin balance changes.
        --- Args: (player: Player, newCoins: number)
        OnCoinsChanged = Signal.new(),
    }, DataService)
end

--- Returns the cached PlayerData for a player.
--- Returns nil if the player's data has not been loaded yet.
--- @param player Player
--- @return PlayerData | nil
function DataService:GetData(player)
    return self._cache[player.UserId]
end

--- Adds coins to a player's balance and fires OnCoinsChanged.
--- @param player Player
--- @param amount number
function DataService:AddCoins(player, amount)
    local data = self._cache[player.UserId]
    if not data then return end
    data.coins = data.coins + amount
    self.OnCoinsChanged:Fire(player, data.coins)
end

--- Loads a player's data from the repository into the cache.
--- Yields until the data source responds.
--- Must be called in PlayerAdded.
--- @param player Player
function DataService:LoadPlayer(player)
    local data = self._repo:Load(player.UserId) or PlayerTypes.Default()
    self._cache[player.UserId] = data
end

--- Saves the player's data to the repository and evicts from cache.
--- Must be called in PlayerRemoving and BindToClose.
--- @param player Player
function DataService:UnloadPlayer(player)
    local data = self._cache[player.UserId]
    if not data then return end
    self._repo:Save(player.UserId, data)
    self._cache[player.UserId] = nil
end

return DataService
```

---

## What is happening here

### The cache

`_cache` is a plain Lua table keyed by `player.UserId`. When a player joins, `LoadPlayer` fetches their data from the Repository and stores it in `_cache`. All reads and mutations happen against this in-memory table. When the player leaves, `UnloadPlayer` saves the cache entry to the Repository and removes it.

This is fast — mutations never hit DataStore. Only `LoadPlayer` and `UnloadPlayer` do.

### The Signal

`OnCoinsChanged` is a public Signal on the Service. Bootstrap will listen to it and forward the new value to the client via NetBridge. The Service does not know anything about the network — it just fires a Signal.

```lua
-- This is all the Service does:
self.OnCoinsChanged:Fire(player, data.coins)

-- Bootstrap handles the network part:
dataSvc.OnCoinsChanged:Connect(function(player, newCoins)
    NetBridge.FireClient("CoinsUpdate", player, newCoins)
end)
```

### LoadPlayer yields

`LoadPlayer` calls `repo:Load()`, which calls `DataStore:GetAsync()`. `GetAsync` is a yielding call — it suspends the current thread until DataStore responds. This is fine because Bootstrap calls `LoadPlayer` inside a `task.spawn`, so it only blocks that player's thread, not the entire server.

---

Next: [Step 5 — Server Bootstrap](05-server-bootstrap.md)
