# Service

Services contain all business logic. They receive a Repository via constructor injection, maintain an in-memory cache, mutate data, and fire Signals when state changes. They never call the network directly and never touch a data source directly.

> **Server-only.** Services run in `ServerScriptService` and are only accessible via the ServiceLocator on the server side.

---

## Template

```lua
--!nostrict

--- @class XService
--- [Describe the domain this service manages.]

local Signal = require(game.ReplicatedStorage.Shared.Framework.Signal)

local XService = {}
XService.__index = XService

--- @param repo XRepository — injected, never constructed internally
--- @return XService
function XService.new(repo)
    return setmetatable({
        _repo  = repo,
        _cache = {},
        --- Fired when something changes. Args: (player: Player, newValue: any)
        OnChanged = Signal.new(),
    }, XService)
end

--- Returns cached data for a player.
--- @param player Player
--- @return table | nil
function XService:GetData(player)
    return self._cache[player.UserId]
end

--- Loads player data from the repository into the cache. Yields.
--- Must be called in PlayerAdded.
--- @param player Player
function XService:LoadPlayer(player)
    local data = self._repo:Load(player.UserId) or DefaultData()
    self._cache[player.UserId] = data
end

--- Saves player data to the repository and evicts from cache.
--- Must be called in PlayerRemoving.
--- @param player Player
function XService:UnloadPlayer(player)
    local data = self._cache[player.UserId]
    if not data then return end
    self._repo:Save(player.UserId, data)
    self._cache[player.UserId] = nil
end

return XService
```

---

## Example A — DataService

Manages coins and the full player data lifecycle.

```lua
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
        --- Fired when a player's coin balance changes. Args: (player: Player, newCoins: number)
        OnCoinsChanged = Signal.new(),
    }, DataService)
end

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

--- Loads from repository into cache. Yields until the data source responds.
--- @param player Player
function DataService:LoadPlayer(player)
    local data = self._repo:Load(player.UserId) or PlayerTypes.Default()
    self._cache[player.UserId] = data
end

--- Saves to repository and evicts from cache.
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

## Example B — InventoryService

Manages item add/remove operations with its own Signal.

```lua
--!nostrict

--- @class InventoryService
--- Manages all player inventory mutations.

local Signal = require(game.ReplicatedStorage.Shared.Framework.Signal)

local InventoryService = {}
InventoryService.__index = InventoryService

--- @param dataService DataService
--- @return InventoryService
function InventoryService.new(dataService)
    return setmetatable({
        _dataSvc = dataService,
        --- Fired on any inventory change. Args: (player: Player, inventory: {[string]: number})
        OnInventoryChanged = Signal.new(),
    }, InventoryService)
end

--- @param player Player
--- @return {[string]: number} | nil
function InventoryService:GetInventory(player)
    local data = self._dataSvc:GetData(player)
    return data and data.inventory
end

--- @param player Player
--- @param itemId string
--- @param amount number?  defaults to 1
function InventoryService:AddItem(player, itemId, amount)
    local data = self._dataSvc:GetData(player)
    if not data then return end
    amount = amount or 1
    data.inventory[itemId] = (data.inventory[itemId] or 0) + amount
    self.OnInventoryChanged:Fire(player, data.inventory)
end

--- @param player Player
--- @param itemId string
--- @param amount number?  defaults to 1
--- @return boolean
function InventoryService:RemoveItem(player, itemId, amount)
    local data = self._dataSvc:GetData(player)
    if not data then return false end
    amount = amount or 1
    local current = data.inventory[itemId] or 0
    if current < amount then return false end
    data.inventory[itemId] = current - amount
    if data.inventory[itemId] == 0 then
        data.inventory[itemId] = nil
    end
    self.OnInventoryChanged:Fire(player, data.inventory)
    return true
end

return InventoryService
```

---

## Key rules

**Services never call NetBridge.** A Service fires a Signal. Bootstrap listens to that Signal and calls `NetBridge.FireClient`. The Service has no knowledge of the network.

```lua
-- ✅ Correct
function DataService:AddCoins(player, amount)
    data.coins += amount
    self.OnCoinsChanged:Fire(player, data.coins)  -- Signal only
end

-- In Bootstrap.server.lua:
dataSvc.OnCoinsChanged:Connect(function(player, newCoins)
    NetBridge.FireClient("CoinsUpdate", player, newCoins)
end)

-- ❌ Wrong
function DataService:AddCoins(player, amount)
    data.coins += amount
    NetBridge.FireClient("CoinsUpdate", player, data.coins)  -- Service knows about network
end
```

**Services never call other Services via ServiceLocator.** If a Service needs another Service, receive it via constructor injection.

```lua
-- ✅ Correct
function InventoryService.new(dataService)
    -- dataService injected by Bootstrap
end

-- ❌ Wrong
function InventoryService.new()
    local dataSvc = ServiceLocator.Get("DataService")  -- hidden dependency
end
```
