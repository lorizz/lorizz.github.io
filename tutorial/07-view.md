# Tutorial — Step 7: View

The `HUDView` renders the coin balance from `HUDCubit` to a `TextLabel` in the ScreenGui. It reads the initial state once, then connects to `OnStateChanged` to update whenever the Cubit emits new state.

---

## HUDView

```lua
-- Client/Views/HUDView.lua
--!nostrict

--- @class HUDView
--- Renders HUDCubit state to the HUD ScreenGui.

local function Mount(gui, cubit)
    local label = gui:WaitForChild("HUD"):WaitForChild("CoinsLabel")

    -- Render the initial state immediately
    label.Text = tostring(cubit.state.coins)

    -- Re-render every time the Cubit emits new state
    cubit.OnStateChanged:Connect(function(state)
        label.Text = tostring(state.coins)
    end)
end

return { Mount = Mount }
```

---

## What is happening here

The View is a module with a single exported function `Mount`. It takes the ScreenGui and the Cubit as parameters — it has no knowledge of ServiceLocator, NetBridge, or any other part of the system.

`WaitForChild` is used because the ScreenGui may not be fully replicated when the LocalScript runs. It yields until the instance exists.

The initial `label.Text = tostring(cubit.state.coins)` renders the current state synchronously. The `OnStateChanged:Connect` renders every subsequent state change. Both read from the same `state` table structure — there is no special handling for the first render vs. updates.

---

## The View has no logic

The View does not decide anything. It does not check `if state.coins > 100`. It does not format the number differently based on some condition. All of that belongs in the Cubit.

If you need to format the coin display, add a method to the Cubit:

```lua
-- In HUDCubit — formatting is logic, so it lives here
function HUDCubit:_formatCoins(coins)
    if coins >= 1000 then
        return string.format("%.1fK", coins / 1000)
    end
    return tostring(coins)
end

function HUDCubit:SetCoins(newCoins)
    self:_emit({ coins = newCoins, coinsDisplay = self:_formatCoins(newCoins) })
end

-- In HUDView — just reads the pre-formatted value
cubit.OnStateChanged:Connect(function(state)
    label.Text = state.coinsDisplay
end)
```

---

Next: [Step 8 — Client Bootstrap](08-client-bootstrap.md)
