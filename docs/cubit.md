# Cubit

A Cubit owns both the UI state and the user-initiated actions for one feature. It extends the base `Cubit` class, defines its state shape, and calls `self:_emit(newState)` whenever something changes. Views observe `OnStateChanged` and re-render.

> **Client-only.** Cubits run in `StarterPlayerScripts`.

---

## How it works

```
User clicks button
  → View calls cubit:DoAction()
      → Cubit calls NetBridge.InvokeServer()  [if needed]
      → Cubit calls self:_emit(newState)
          → OnStateChanged:Fire(newState)
              → View re-renders
```

The View never calls NetBridge directly. The Cubit never touches GUI elements. The boundary is clean.

---

## Template

```lua
--!nonstrict

--- @class XCubit
--- Manages [feature] state and actions.
--- @field state { ... }

local Cubit     = require(game.ReplicatedStorage.Shared.Framework.Cubit)
local NetBridge = require(game.ReplicatedStorage.Shared.Framework.NetBridge)

local XCubit = {}
XCubit.__index = XCubit
setmetatable(XCubit, { __index = Cubit })

--- @param initialState table
--- @return XCubit
function XCubit.new(initialState)
    return setmetatable(
        Cubit.new(initialState),
        XCubit
    )
end

--- Called from Bootstrap when the server pushes new data.
--- @param newValue any
function XCubit:SetX(newValue)
    self:_emit({ x = newValue })
end

--- User-initiated action. Called from a View.
function XCubit:DoAction()
    local result = NetBridge.InvokeServer("ActionName")
    if result.ok then
        self:_emit({ x = result.newValue })
    end
end

return XCubit
```

---

## Example A — HUDCubit

Simple Cubit that only receives server pushes. No user actions.

```lua
--!nonstrict

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

--- Called when the server pushes a coin update.
--- @param newCoins number
function HUDCubit:SetCoins(newCoins)
    self:_emit({ coins = newCoins })
end

return HUDCubit
```

---

## Example B — InventoryCubit with optimistic update

Cubit that handles both server pushes and a user action, with optimistic UI update and rollback.

```lua
--!nonstrict

--- @class InventoryCubit
--- Manages inventory state and the UseOrb action.
--- @field state { inventory: {[string]: number} }

local Cubit     = require(game.ReplicatedStorage.Shared.Framework.Cubit)
local NetBridge = require(game.ReplicatedStorage.Shared.Framework.NetBridge)

local InventoryCubit = {}
InventoryCubit.__index = InventoryCubit
setmetatable(InventoryCubit, { __index = Cubit })

--- @param initialInventory {[string]: number}
--- @return InventoryCubit
function InventoryCubit.new(initialInventory)
    return setmetatable(
        Cubit.new({ inventory = initialInventory }),
        InventoryCubit
    )
end

--- Called when the server pushes an inventory update.
--- @param inventory {[string]: number}
function InventoryCubit:SetInventory(inventory)
    self:_emit({ inventory = inventory })
end

--- Uses one orb. Applies an optimistic update immediately,
--- then rolls back if the server rejects the request.
function InventoryCubit:UseOrb()
    local current = self.state.inventory["orb"] or 0
    if current <= 0 then return end

    -- Optimistic: show the result before the server confirms
    local optimistic = table.clone(self.state.inventory)
    optimistic["orb"] = current - 1
    if optimistic["orb"] == 0 then optimistic["orb"] = nil end
    self:_emit({ inventory = optimistic })

    local result = NetBridge.InvokeServer("UseOrb")

    if not result.ok then
        -- Rollback: server rejected, restore previous state
        self:_emit({ inventory = self.state.inventory })
        warn("[InventoryCubit] UseOrb rejected:", result.reason)
    end
end

return InventoryCubit
```

---

## Example C — ShopCubit (open/close + purchase)

```lua
--!nonstrict

--- @class ShopCubit
--- Manages shop visibility and purchase flow.
--- @field state { isOpen: boolean, lastError: string? }

local Cubit     = require(game.ReplicatedStorage.Shared.Framework.Cubit)
local NetBridge = require(game.ReplicatedStorage.Shared.Framework.NetBridge)

local ShopCubit = {}
ShopCubit.__index = ShopCubit
setmetatable(ShopCubit, { __index = Cubit })

--- @return ShopCubit
function ShopCubit.new()
    return setmetatable(
        Cubit.new({ isOpen = false, lastError = nil }),
        ShopCubit
    )
end

function ShopCubit:Open()
    self:_emit({ isOpen = true, lastError = nil })
end

function ShopCubit:Close()
    self:_emit({ isOpen = false, lastError = nil })
end

--- @param itemId string
function ShopCubit:Purchase(itemId)
    local result = NetBridge.InvokeServer("PurchaseItem", itemId)
    if not result.ok then
        self:_emit({ isOpen = self.state.isOpen, lastError = result.reason })
    end
end

return ShopCubit
```

---

## Cubits talking to each other

When one Cubit needs to react to another Cubit's state changes, use `ServiceLocator` internally. Never pass Cubits to each other via the constructor — that would create coupling in Bootstrap.

```lua
--!nonstrict

--- @class NotificationCubit
--- Listens to other Cubits and surfaces notification messages.
--- @field state { messages: {string} }

local Cubit          = require(game.ReplicatedStorage.Shared.Framework.Cubit)
local ServiceLocator = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)

local NotificationCubit = {}
NotificationCubit.__index = NotificationCubit
setmetatable(NotificationCubit, { __index = Cubit })

--- Note: InventoryCubit must be registered in ServiceLocator before this is called.
--- @return NotificationCubit
function NotificationCubit.new()
    local self = setmetatable(
        Cubit.new({ messages = {} }),
        NotificationCubit
    )

    local invCubit = ServiceLocator.Get("InventoryCubit")
    invCubit.OnStateChanged:Connect(function(state)
        if (state.inventory["orb"] or 0) >= 10 then
            self:_push("You have 10 orbs — use them!")
        end
    end)

    return self
end

--- @param message string
function NotificationCubit:_push(message)
    local updated = table.clone(self.state.messages)
    table.insert(updated, message)
    self:_emit({ messages = updated })
end

return NotificationCubit
```

In Bootstrap, register `InventoryCubit` first:

```lua
local invCubit   = InventoryCubit.new({})
ServiceLocator.Register("InventoryCubit", invCubit)

-- NotificationCubit resolves InventoryCubit internally
local notifCubit = NotificationCubit.new()
ServiceLocator.Register("NotificationCubit", notifCubit)
```

---

## Key rules

**Views never call `_emit` directly.** `_emit` is a protected method for Cubit subclasses only.

**One Cubit per feature, not per screen.** A feature is a cohesive unit of state and actions. A screen may mount multiple Cubits.

**`self.state` is read-only outside of `_emit`.** Never mutate `self.state` directly — always go through `self:_emit(newState)`.
