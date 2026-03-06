# Folder Structure

Every folder has a single purpose. If you are unsure where a file belongs, the answer is here.

---

## Full Structure

```
ReplicatedStorage/
└── Shared/
    ├── Framework/                   ← copy once, never touch
    │   ├── Signal.lua
    │   ├── ServiceLocator.lua
    │   ├── Context.lua
    │   ├── Cubit.lua
    │   └── NetBridge.lua
    └── Types/                       ← shared data shapes
        └── PlayerTypes.lua

ServerScriptService/
└── Server/
    ├── Bootstrap.server.lua         ← server composition root
    ├── Repositories/                ← data source abstraction
    │   └── PlayerRepository.lua
    ├── Services/                    ← business logic
    │   └── DataService.lua
    └── Controllers/                 ← incoming client requests
        └── PlayerController.lua

StarterPlayerScripts/
└── Client/
    ├── Bootstrap.client.lua         ← client composition root
    ├── Cubits/                      ← state + actions per feature
    │   └── HUDCubit.lua
    └── Views/                       ← pure render per feature
        └── HUDView.lua
```

---

## Folder Responsibilities

### `Shared/Framework/`

The five core framework files. You copy them once at project setup and never touch them again. They have no game-specific logic — they are pure infrastructure.

### `Shared/Types/`

All data shape definitions shared between server and client. A type file is just a module that returns a `Default()` function and documents the shape of a table.

```lua
-- Shared/Types/PlayerTypes.lua
local function Default()
    return { coins = 0, inventory = {} }
end
return { Default = Default }
```

### `Server/Repositories/`

One file per data source. A Repository exposes `Load` and `Save` and nothing else. The implementation behind those methods is entirely up to the file — DataStore, HTTP, memory.

### `Server/Services/`

One file per domain (e.g. `DataService`, `InventoryService`, `ShopService`). A Service holds an in-memory cache, exposes mutation methods, and fires Signals when state changes.

### `Server/Controllers/`

One file per domain, mirroring Services. A Controller registers `NetBridge.OnServer` handlers in its `Init()` method, validates input, and delegates to Services.

### `Client/Cubits/`

One file per UI feature. A Cubit extends the base `Cubit` class, defines its state shape, and exposes methods that either update state locally or call `NetBridge.InvokeServer`.

### `Client/Views/`

One file per UI feature. A View is a module with a single `Mount(gui, cubit)` function. It reads initial state from the Cubit, connects to `cubit.OnStateChanged`, and wires up button clicks.

---

## Adding a New Feature

A new feature — for example a Shop — requires exactly these files:

```
Server/Services/ShopService.lua       ← purchase validation, price lookup
Server/Controllers/ShopController.lua ← NetBridge.OnServer("PurchaseItem", ...)
Client/Cubits/ShopCubit.lua           ← isOpen state, Purchase() action
Client/Views/ShopView.lua             ← renders items, wires buy buttons
```

Then in the two Bootstrap files:

```lua
-- Bootstrap.server.lua
local shopSvc = ShopService.new(dataSvc, invSvc)
ServiceLocator.Register("ShopService", shopSvc)
NetBridge.CreateEvent("ShopUpdate")     -- if server needs to push to client
ShopController.Init()

-- Bootstrap.client.lua
local shopCubit = ShopCubit.new()
ServiceLocator.Register("ShopCubit", shopCubit)
ShopView.Mount(gui, shopCubit)
```

---

## Rules

> **Never put a Repository on the client.** Repositories call DataStore, HttpService, or other server-only APIs. They must never be required from a LocalScript or a Cubit.

> **Never put a Script inside a Service or Repository file.** They are `ModuleScript` instances, not `Script` instances.

> **The only Scripts in the project are the two Bootstrap files.** Everything else is a `ModuleScript`.
