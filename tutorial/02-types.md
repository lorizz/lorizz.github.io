# Tutorial — Step 2: Types

Type files define the shape of your data. They live in `Shared/Types/` so both the server and client can import them. They are plain modules — no classes, no logic beyond a `Default()` function that returns a fresh table.

---

## PlayerTypes

`PlayerData` is the only persistent data structure in this game. It holds the player's coin balance.

```lua
-- Shared/Types/PlayerTypes.lua
--!nonstrict

--- @type PlayerData
--- The full persistent state for a single player.
--- { coins: number }

--- Returns a default PlayerData table for a new player.
--- @return PlayerData
local function Default()
    return {
        coins = 0,
    }
end

return { Default = Default }
```

---

## Why a Default() function?

When a player joins for the first time, `PlayerRepository:Load()` returns `nil` (no existing data). The Service uses `Default()` as the fallback:

```lua
local data = self._repo:Load(player.UserId) or PlayerTypes.Default()
```

This ensures every player always has a valid data table in the cache, whether they are new or returning.

---

## Extending PlayerData

Later in the tutorial series, we will add an `inventory` field. When you need to extend `PlayerData`, add the field here and update `Default()`:

```lua
local function Default()
    return {
        coins     = 0,
        inventory = {},      -- added later
    }
end
```

Nothing else in the architecture changes. The Repository persists whatever is in the table. The Service mutates it. The Cubit reflects it.

---

Next: [Step 3 — Repository](03-repository.md)
