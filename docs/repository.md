# Repository

A Repository abstracts a data source. It exposes a fixed interface — `Load` and `Save` — while hiding the implementation entirely. Services call these methods without knowing or caring what is behind them.

> **Server-only.** Never require a Repository from a LocalScript or a Cubit. Repositories may call DataStore, HttpService, or other server-only APIs.

---

## Template

```lua
--!nonstrict

--- @class XRepository
--- Abstracts [describe your data source].
--- Interface: Load(id) -> data | nil,  Save(id, data)

local XRepository = {}
XRepository.__index = XRepository

--- @return XRepository
function XRepository.new()
    return setmetatable({
        -- initialize your data source connection here
    }, XRepository)
end

--- Loads data for a given id. Returns nil if not found or on error.
--- @param id any
--- @return table | nil
function XRepository:Load(id)
    -- read from your data source
end

--- Persists data for a given id.
--- @param id any
--- @param data table
function XRepository:Save(id, data)
    -- write to your data source
end

return XRepository
```

---

## Example A — DataStore

The most common implementation. Uses `DataStoreService` to persist player data across sessions.

```lua
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

## Example B — In-memory mock

Used during development or testing. Identical interface, no DataStore calls, instant response.

```lua
--!nonstrict

--- @class MockPlayerRepository
--- In-memory implementation. Swap in Bootstrap.server.lua for development.
--- Zero changes required in any Service.

local MockPlayerRepository = {}
MockPlayerRepository.__index = MockPlayerRepository

--- @return MockPlayerRepository
function MockPlayerRepository.new()
    return setmetatable({ _data = {} }, MockPlayerRepository)
end

--- @param userId number
--- @return PlayerData | nil
function MockPlayerRepository:Load(userId)
    return self._data[userId]
end

--- @param userId number
--- @param data PlayerData
function MockPlayerRepository:Save(userId, data)
    self._data[userId] = data
end

return MockPlayerRepository
```

To swap implementations, change a single line in `Bootstrap.server.lua`:

```lua
-- Production
local repo = PlayerRepository.new()

-- Development / testing
local repo = MockPlayerRepository.new()
```

Every Service that depends on `repo` works identically. Nothing else changes.

---

## Example C — HTTP (Supabase)

For games that need a real backend. Same interface, different implementation.

```lua
--!nonstrict

--- @class SupabasePlayerRepository
--- Persists player data via HttpService to a Supabase backend.

local HttpService = game:GetService("HttpService")

local BASE_URL = "https://your-project.supabase.co/rest/v1/players"
local API_KEY  = "your-anon-key"

local SupabasePlayerRepository = {}
SupabasePlayerRepository.__index = SupabasePlayerRepository

--- @return SupabasePlayerRepository
function SupabasePlayerRepository.new()
    return setmetatable({}, SupabasePlayerRepository)
end

--- @param userId number
--- @return PlayerData | nil
function SupabasePlayerRepository:Load(userId)
    local ok, response = pcall(function()
        return HttpService:RequestAsync({
            Url     = BASE_URL .. "?userId=eq." .. userId,
            Method  = "GET",
            Headers = {
                ["apikey"]        = API_KEY,
                ["Authorization"] = "Bearer " .. API_KEY,
            },
        })
    end)
    if not ok or not response.Success then
        warn("[SupabasePlayerRepository] Load failed:", response)
        return nil
    end
    local rows = HttpService:JSONDecode(response.Body)
    return rows[1] and rows[1].data or nil
end

--- @param userId number
--- @param data PlayerData
function SupabasePlayerRepository:Save(userId, data)
    pcall(function()
        HttpService:RequestAsync({
            Url     = BASE_URL,
            Method  = "POST",
            Headers = {
                ["apikey"]        = API_KEY,
                ["Authorization"] = "Bearer " .. API_KEY,
                ["Content-Type"]  = "application/json",
                ["Prefer"]        = "resolution=merge-duplicates",
            },
            Body = HttpService:JSONEncode({ userId = userId, data = data }),
        })
    end)
end

return SupabasePlayerRepository
```

---

## Registering in Bootstrap

```lua
-- Bootstrap.server.lua
local PlayerRepository = require(script.Parent.Repositories.PlayerRepository)

local repo = PlayerRepository.new()
-- repo is injected into Services directly, not registered in ServiceLocator
-- (only Services and Cubits go in ServiceLocator)
local dataSvc = DataService.new(repo)
```

> Repositories are injected directly into Services via constructor. They are not registered in the ServiceLocator — Services are. This prevents any accidental cross-layer access.
