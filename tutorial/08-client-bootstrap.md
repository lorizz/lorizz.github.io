# Tutorial — Step 8: Client Bootstrap

The client Bootstrap is a `LocalScript` in `StarterPlayerScripts/Client/`. It constructs the Cubits, mounts the Views, and connects the NetBridge listeners that drive the Cubits with server data.

---

## Bootstrap.client.lua

```lua
-- Client/Bootstrap.client.lua
--!nonstrict
local ServiceLocator = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)
local NetBridge      = require(game.ReplicatedStorage.Shared.Framework.NetBridge)
local Context        = require(game.ReplicatedStorage.Shared.Framework.Context)
local HUDCubit       = require(script.Parent.Cubits.HUDCubit)
local HUDView        = require(script.Parent.Views.HUDView)

local player = Context.GetPlayer()
local gui    = player.PlayerGui:WaitForChild("MainGui")

-- 1. Construct Cubits with initial state
local hudCubit = HUDCubit.new(0)

-- 2. Register in ServiceLocator (other Cubits may need to resolve this later)
ServiceLocator.Register("HUDCubit", hudCubit)

-- 3. Mount Views — Cubit passed explicitly, not fetched inside the View
HUDView.Mount(gui, hudCubit)

-- 4. Server → Cubit pipeline
-- When the server fires "CoinsUpdate", update the Cubit.
-- The Cubit emits new state. The View re-renders automatically.
NetBridge.OnClient("CoinsUpdate", function(newCoins)
    hudCubit:SetCoins(newCoins)
end)
```

---

## The full data pipeline

This is the complete path from a coin being added on the server to the label updating on the client:

```
[Server — every 1 second]
  dataSvc:AddCoins(player, 1)
    → _cache[userId].coins += 1
    → OnCoinsChanged:Fire(player, newCoins)        -- Signal (in-process)

[Server Bootstrap — Signal listener]
  dataSvc.OnCoinsChanged:Connect(...)
    → NetBridge.FireClient("CoinsUpdate", player, newCoins)

[Client — NetBridge listener]
  NetBridge.OnClient("CoinsUpdate", function(newCoins)
    hudCubit:SetCoins(newCoins)                    -- Cubit method
      → self:_emit({ coins = newCoins })
          → OnStateChanged:Fire(state)             -- Signal (in-process)

[HUDView — Signal listener]
  cubit.OnStateChanged:Connect(function(state)
    label.Text = tostring(state.coins)             -- only line that touches GUI
  end)
```

Every layer has one job. The Service mutates data and signals. Bootstrap bridges signals to the network. NetBridge delivers to the client. The Cubit holds state. The View renders.

---

## Why `Context.GetPlayer()`

`Context.GetPlayer()` is shorthand for `game.Players.LocalPlayer`. In a LocalScript, `LocalPlayer` is always available, so this is purely a convenience. Its value is that every client-side file uses the same one-line pattern instead of `game:GetService("Players").LocalPlayer`.

---

Next: [Step 9 — Result](09-result.md)
