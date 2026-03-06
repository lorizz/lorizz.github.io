# Server Controller

A Server Controller is the entry point for client requests. It registers `NetBridge.OnServer` handlers in a single `Init()` method, validates input, and delegates to Services. It contains zero business logic — if you find yourself writing game rules here, they belong in a Service.

> **Server-only.** Controllers run on the server and resolve Services from the ServiceLocator.

---

## Template

```lua
--!nostrict

--- @class XController
--- Handles incoming client requests for [domain].
--- Validates input. Delegates to Services. No business logic.

local ServiceLocator = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)
local NetBridge      = require(game.ReplicatedStorage.Shared.Framework.NetBridge)

local XController = {}

--- Registers all server-side handlers. Call once from Bootstrap.server.lua.
function XController.Init()

    NetBridge.OnServer("ActionName", function(player, arg1, arg2)
        -- 1. validate input
        if type(arg1) ~= "string" then
            return { ok = false, reason = "Invalid input" }
        end

        -- 2. get the service
        local xSvc = ServiceLocator.Get("XService")

        -- 3. delegate and return result
        return xSvc:DoSomething(player, arg1, arg2)
    end)

end

return XController
```

---

## Example — PlayerController

```lua
--!nostrict

--- @class PlayerController
--- Handles all incoming client requests related to player actions.

local ServiceLocator = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)
local NetBridge      = require(game.ReplicatedStorage.Shared.Framework.NetBridge)

local PlayerController = {}

function PlayerController.Init()

    NetBridge.OnServer("UseOrb", function(player)
        local invSvc  = ServiceLocator.Get("InventoryService")
        local dataSvc = ServiceLocator.Get("DataService")

        local inv = invSvc:GetInventory(player)
        if not inv or (inv["orb"] or 0) <= 0 then
            return { ok = false, reason = "No orbs" }
        end

        invSvc:RemoveItem(player, "orb", 1)
        dataSvc:AddCoins(player, 50)

        return { ok = true }
    end)

    NetBridge.OnServer("PurchaseItem", function(player, itemId)
        if type(itemId) ~= "string" or itemId == "" then
            return { ok = false, reason = "Invalid itemId" }
        end

        local shopSvc = ServiceLocator.Get("ShopService")
        return shopSvc:Purchase(player, itemId)
    end)

end

return PlayerController
```

---

## Calling from the Client

The client calls a Controller endpoint via `NetBridge.InvokeServer`. The result is the table returned by the handler.

```lua
-- Client/Cubits/InventoryCubit.lua
function InventoryCubit:UseOrb()
    local result = NetBridge.InvokeServer("UseOrb")
    if not result.ok then
        warn("[InventoryCubit] UseOrb failed:", result.reason)
    end
end
```

---

## Response convention

Always return a table with an `ok` boolean. On failure include a `reason` string. This gives the client a consistent structure to check.

```lua
-- Success
return { ok = true }

-- Failure
return { ok = false, reason = "Not enough coins" }

-- Success with payload
return { ok = true, newBalance = 150 }
```

---

## Key rules

**No business logic in Controllers.** The Controller's job is to receive, validate, and delegate. If you are writing `if data.coins > price then` in a Controller, that code belongs in a Service.

**One `Init()` per Controller.** All handlers for a domain are registered together in one `Init()` call. Never split handlers for the same domain across multiple files.

**Controllers are not classes.** They have no `new()`, no `__index`, no state. They are plain modules with a single `Init()` function.
