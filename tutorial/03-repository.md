# Tutorial — Step 3: Repository

The Repository abstracts DataStore. `DataService` will call `repo:Load()` and `repo:Save()` — it will never touch `DataStoreService` directly.

---

## PlayerRepository

```lua
-- Server/Repositories/PlayerRepository.lua
--!nonstrict

--- @class PlayerRepository
--- Persists player data using DataStoreService.

local DataStoreService = game:GetService("DataStoreService")
local PlayerTypes      = require(game.ReplicatedStorage.Shared.Types.PlayerTypes)

local STORE_KEY = "PlayerData_v1"

local PlayerRepository = {}
PlayerRepository.__index = PlayerRepository

--- @return PlayerRepository
function PlayerRepository.new()
    return setmetatable({
        _store = DataStoreService:GetDataStore(STORE_KEY),
    }, PlayerRepository)
end

--- Loads a player's data from DataStore.
--- Returns nil on error or if no data exists yet.
--- @param userId number
--- @return PlayerData | nil
function PlayerRepository:Load(userId)
    local ok, data = pcall(function()
        return self._store:GetAsync(tostring(userId))
    end)
    if not ok then
        warn(("[PlayerRepository] Load failed for %d: %s"):format(userId, data))
        return nil
    end
    return data
end

--- Persists a player's data to DataStore.
--- @param userId number
--- @param data PlayerData
function PlayerRepository:Save(userId, data)
    local ok, err = pcall(function()
        self._store:SetAsync(tostring(userId), data)
    end)
    if not ok then
        warn(("[PlayerRepository] Save failed for %d: %s"):format(userId, err))
    end
end

return PlayerRepository
```

---

## What is happening here

`GetAsync` and `SetAsync` are wrapped in `pcall` because DataStore calls can fail — network issues, rate limits, Studio sandbox restrictions. The Repository handles the error silently (a `warn`) and returns `nil` on load failure. The Service is responsible for handling a `nil` result by falling back to `Default()`.

The key used in DataStore is `tostring(userId)` — DataStore keys must be strings, and `userId` is a number.

`STORE_KEY = "PlayerData_v1"` includes a version suffix. If you ever change the shape of `PlayerData` in a breaking way, bump this to `"PlayerData_v2"` to start fresh without corrupting existing data.

---

## Testing without DataStore

During development you can swap in the mock Repository to avoid DataStore rate limits. Change one line in Bootstrap later:

```lua
-- Development: instant, no DataStore calls
local repo = MockPlayerRepository.new()

-- Production
local repo = PlayerRepository.new()
```

The mock looks like this — identical interface, plain table storage:

```lua
-- (optional) Server/Repositories/MockPlayerRepository.lua
--!nonstrict

local MockPlayerRepository = {}
MockPlayerRepository.__index = MockPlayerRepository

function MockPlayerRepository.new()
    return setmetatable({ _data = {} }, MockPlayerRepository)
end

function MockPlayerRepository:Load(userId)
    return self._data[userId]
end

function MockPlayerRepository:Save(userId, data)
    self._data[userId] = data
end

return MockPlayerRepository
```

---

Next: [Step 4 — Service](04-service.md)
