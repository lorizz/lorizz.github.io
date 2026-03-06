# Tutorial — Step 9: Result

You have built a complete coin game using the framework. Here is a recap of every file and what it does.

---

## Complete file list

```
ReplicatedStorage/Shared/Framework/Signal.lua
ReplicatedStorage/Shared/Framework/ServiceLocator.lua
ReplicatedStorage/Shared/Framework/Context.lua
ReplicatedStorage/Shared/Framework/Cubit.lua
ReplicatedStorage/Shared/Framework/NetBridge.lua

ReplicatedStorage/Shared/Types/PlayerTypes.lua

Server/Repositories/PlayerRepository.lua
Server/Services/DataService.lua
Server/Bootstrap.server.lua

Client/Cubits/HUDCubit.lua
Client/Views/HUDView.lua
Client/Bootstrap.client.lua
```

12 files total. 5 are the framework (copy once, never touch). 7 are game-specific.

---

## What each file does

| File | What it does |
|---|---|
| `PlayerTypes.lua` | Defines `{ coins: number }` and `Default()` |
| `PlayerRepository.lua` | Calls `DataStore:GetAsync` and `SetAsync` |
| `DataService.lua` | Holds the coin cache, exposes `AddCoins`, fires `OnCoinsChanged` |
| `Bootstrap.server.lua` | Wires everything: constructs, registers, connects signals, manages lifecycle |
| `HUDCubit.lua` | Holds `{ coins: number }` on the client, exposes `SetCoins` |
| `HUDView.lua` | Connects `CoinsLabel.Text` to `HUDCubit.OnStateChanged` |
| `Bootstrap.client.lua` | Constructs Cubits, mounts Views, connects NetBridge to Cubits |

---

## Run it

1. Press **Play** in Roblox Studio (use the server + client test mode with at least one player)
2. The `CoinsLabel` should show `0` immediately on join
3. Within one second it should update to the player's actual coin count (0 for a new player)
4. Every second the label increments by 1
5. Press **Stop**, then **Play** again — the label should start from where you left off

If coins reset to 0 on rejoin, check that `BindToClose` is present in `Bootstrap.server.lua` and that you are testing with the multi-player Studio test mode (not single-player), as DataStore requires a published game or Studio with API access enabled.

---

## Enabling DataStore in Studio

DataStore calls require API access to be enabled in Studio:

1. Go to **File → Game Settings → Security**
2. Enable **Allow Studio Access to API Services**

Without this, all `GetAsync` calls return `nil` (no error, just no data), and `SetAsync` silently fails.

---

## What to build next

Now that the foundation is working, the architecture scales naturally:

**Add an inventory system** — create `InventoryService`, add `inventory` to `PlayerTypes`, create `InventoryCubit` and `InventoryView`.

**Add collectible parts** — create `ItemSpawner` in `Server/Services/`, touch detection delegates to `InventoryService:AddItem()`.

**Add a shop** — create `ShopService` on the server, `ShopCubit` and `ShopView` on the client, one new `NetBridge.OnServer("PurchaseItem", ...)` in a `ShopController`.

Each new feature follows the exact same pattern. The only files that change are Bootstrap (to register the new classes) and the new feature files themselves.

---

## Common issues

**`CoinsLabel` not found** — check that the GUI path is exactly `MainGui > HUD > CoinsLabel` and that all three instances exist in StarterGui.

**Coins always start at 0** — DataStore API access is not enabled in Studio. See above.

**`Service 'DataService' not found` error** — `ServiceLocator.Register("DataService", dataSvc)` is missing or has a typo in `Bootstrap.server.lua`.

**`NetBridge: RemoteEvent 'CoinsUpdate' not found` error** — `NetBridge.CreateEvent("CoinsUpdate")` was not called in the server Bootstrap, or the client Bootstrap ran before the server Bootstrap created the event. Ensure `CreateEvent` is called at the top of the server Bootstrap, not inside a callback.
