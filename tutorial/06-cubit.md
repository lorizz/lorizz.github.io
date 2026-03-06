# Tutorial — Step 6: Cubit

The `HUDCubit` manages the coin balance on the client. It holds the current state and exposes a `SetCoins` method that the Bootstrap calls when the server pushes an update. The View observes `OnStateChanged` and re-renders whenever the state changes.

---

## HUDCubit

```lua
-- Client/Cubits/HUDCubit.lua
--!nostrict

--- @class HUDCubit
--- Manages HUD state. Receives coin updates pushed from the server.
--- @field state { coins: number }

local Cubit = require(game.ReplicatedStorage.Shared.Framework.Cubit)

local HUDCubit = {}
HUDCubit.__index = HUDCubit
setmetatable(HUDCubit, { __index = Cubit })

--- @param initialCoins number
--- @return HUDCubit
function HUDCubit.new(initialCoins)
    return setmetatable(
        Cubit.new({ coins = initialCoins }),
        HUDCubit
    )
end

--- Called when the server pushes a new coin balance.
--- @param newCoins number
function HUDCubit:SetCoins(newCoins)
    self:_emit({ coins = newCoins })
end

return HUDCubit
```

---

## What is happening here

### Extending Cubit

`setmetatable(HUDCubit, { __index = Cubit })` sets up inheritance. When Lua looks for a method on `HUDCubit` and does not find it, it falls through to `Cubit`. This is how `self:_emit()` is available even though `HUDCubit` does not define it.

### State shape

The state is always a plain table: `{ coins = number }`. When `SetCoins` is called, `_emit` replaces the entire state table and fires `OnStateChanged`. The View receives the new state table and reads `state.coins` from it.

### Initial state

`HUDCubit.new(0)` creates the Cubit with `{ coins = 0 }`. The real value arrives shortly after from the server via `NetBridge.OnClient("CoinsUpdate", ...)`. The View shows 0 for a brief moment, then the server pushes the real value and the View updates.

---

## When a Cubit does more

In this tutorial `HUDCubit` only receives server pushes — there is no user action. In a more complex feature, the Cubit would also call `NetBridge.InvokeServer`:

```lua
-- Example: a Cubit with a user action
function ShopCubit:Purchase(itemId)
    local result = NetBridge.InvokeServer("PurchaseItem", itemId)
    if not result.ok then
        self:_emit({ isOpen = self.state.isOpen, lastError = result.reason })
    end
end
```

The pattern is always the same: user triggers a Cubit method → Cubit optionally calls the server → Cubit calls `_emit` with the new state.

---

Next: [Step 7 — View](07-view.md)
