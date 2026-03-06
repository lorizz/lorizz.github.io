# View

A View is a pure render function. It receives a Cubit as a parameter, reads its initial state, connects to `OnStateChanged`, and updates GUI elements. It contains no logic, makes no network calls, and does not use the ServiceLocator.

> **Client-only.** Views run in `StarterPlayerScripts`.

---

## Template

```lua
--!nonstrict

--- @class XView
--- Renders XCubit state to ScreenGui elements.
--- Forwards user input to the Cubit.

--- Mounts the view. Binds all GUI elements to the Cubit.
--- @param gui ScreenGui
--- @param cubit XCubit
local function Mount(gui, cubit)
    local someLabel  = gui:WaitForChild("Frame"):WaitForChild("Label")
    local someButton = gui:WaitForChild("Frame"):WaitForChild("Button")

    -- Render initial state
    someLabel.Text = tostring(cubit.state.someValue)

    -- React to state changes
    cubit.OnStateChanged:Connect(function(state)
        someLabel.Text = tostring(state.someValue)
    end)

    -- Forward input to Cubit — never to NetBridge directly
    someButton.MouseButton1Click:Connect(function()
        cubit:DoAction()
    end)
end

return { Mount = Mount }
```

---

## Example A — HUDView

Simple view with no user input. Only renders state.

```lua
--!nonstrict

--- @class HUDView
--- Renders HUDCubit state to the HUD ScreenGui.

local function Mount(gui, cubit)
    local label = gui:WaitForChild("HUD"):WaitForChild("CoinsLabel")

    label.Text = tostring(cubit.state.coins)

    cubit.OnStateChanged:Connect(function(state)
        label.Text = tostring(state.coins)
    end)
end

return { Mount = Mount }
```

---

## Example B — InventoryView

View that renders a dynamic list and handles a button click.

```lua
--!nonstrict

--- @class InventoryView
--- Renders InventoryCubit state and forwards UseOrb input to the Cubit.

local function Mount(gui, cubit)
    local list         = gui:WaitForChild("Inventory"):WaitForChild("List")
    local useOrbButton = gui:WaitForChild("Inventory"):WaitForChild("UseOrbButton")

    local function render(state)
        for _, child in list:GetChildren() do
            if child:IsA("TextLabel") then child:Destroy() end
        end
        for itemId, count in state.inventory do
            local label                     = Instance.new("TextLabel")
            label.Size                      = UDim2.new(1, 0, 0, 24)
            label.Text                      = itemId .. " x" .. count
            label.BackgroundTransparency    = 1
            label.Parent                    = list
        end
    end

    render(cubit.state)

    cubit.OnStateChanged:Connect(render)

    useOrbButton.MouseButton1Click:Connect(function()
        cubit:UseOrb()
    end)
end

return { Mount = Mount }
```

---

## Example C — ShopView

View that toggles visibility and surfaces an error message.

```lua
--!nonstrict

--- @class ShopView
--- Renders ShopCubit state and forwards purchase input.

local function Mount(gui, cubit)
    local shopFrame   = gui:WaitForChild("Shop")
    local errorLabel  = shopFrame:WaitForChild("ErrorLabel")
    local closeButton = shopFrame:WaitForChild("CloseButton")
    local swordButton = shopFrame:WaitForChild("BuySword")

    cubit.OnStateChanged:Connect(function(state)
        shopFrame.Visible  = state.isOpen
        errorLabel.Text    = state.lastError or ""
        errorLabel.Visible = state.lastError ~= nil
    end)

    closeButton.MouseButton1Click:Connect(function()
        cubit:Close()
    end)

    swordButton.MouseButton1Click:Connect(function()
        cubit:Purchase("sword")
    end)
end

return { Mount = Mount }
```

---

## When to use Unmount

For views that are dynamically mounted and unmounted (e.g. a popup that opens and closes), store connections and disconnect them on unmount.

```lua
--!nonstrict

--- @class PopupView
--- A view that can be mounted and unmounted dynamically.

local function Mount(gui, cubit)
    local frame  = gui:WaitForChild("Popup")
    local connections = {}

    local function render(state)
        frame.Visible = state.isOpen
    end

    render(cubit.state)
    table.insert(connections, cubit.OnStateChanged:Connect(render))

    local function Unmount()
        for _, conn in connections do conn:Disconnect() end
        table.clear(connections)
    end

    return { Unmount = Unmount }
end

return { Mount = Mount }
```

For permanent views like the HUD, `Unmount` is unnecessary.

---

## Key rules

**Views never call NetBridge.** If a button should send something to the server, the View calls a Cubit method. The Cubit calls NetBridge.

**Views never use ServiceLocator.** The Cubit is passed explicitly from Bootstrap. This makes Views independently testable.

**Views never contain conditions about game logic.** `if player.coins > 100` is game logic — it belongs in a Cubit or Service. The View only reads `state.someValue` and renders it.

**Re-render the full relevant section on state change.** Do not try to diff. Destroy and rebuild the relevant elements. For most Roblox UIs this is fast enough and keeps the View simple.
