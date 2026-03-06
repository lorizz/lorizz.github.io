# Core Concepts

Eight layers. Each has a single, clearly defined job. Nothing bleeds into another.

---

## The Eight Layers

| Layer | Side | Responsibility |
|---|---|---|
| **Repository** | Server | Abstracts the data source. The rest of the codebase never knows or cares where data comes from — DataStore, Supabase, in-memory mock, anything. |
| **Service** | Server | Business logic. Mutates data, fires Signals. Never touches the network or a data source directly. |
| **Controller** | Server | Receives client requests via NetBridge, validates input, delegates to Services. Contains zero logic. |
| **Cubit** | Client | Owns UI state and user-initiated actions. The single client-side class per feature. Replaces ViewModel + Controller. |
| **View** | Client | Pure render. Reads Cubit state, updates GUI elements. No logic, no network calls, no ServiceLocator. |
| **NetBridge** | Shared | All remote communication. Never use RemoteEvent or RemoteFunction directly. |
| **ServiceLocator** | Shared | DI container. Register once in Bootstrap, resolve anywhere via `Get()`. |
| **Context** | Client | `GetPlayer()`, `GetCharacter()`, `GetService()` — zero-boilerplate access to common values. |

---

## The Repository Contract

A Repository is an **abstraction over a data source**. The interface — the methods it exposes — never changes. The implementation behind it can be anything.

```
-- Today
PlayerRepository  →  DataStoreService:GetAsync()

-- Tomorrow
PlayerRepository  →  HttpService:RequestAsync()  →  Supabase

-- In tests
PlayerRepository  →  plain Lua table (mock)
```

The Services that call `repo:Load()` and `repo:Save()` never change. Only the Repository file changes. This is the entire value of the pattern.

---

## Cubit vs MVVM

Traditional MVVM splits client-side logic into a ViewModel (holds state) and a Controller (handles actions). Cubit collapses both into a single class — one file per feature, no indirection.

| | MVVM | Cubit |
|---|---|---|
| State | ViewModel | Cubit |
| User actions | Controller | Cubit |
| View gets dependencies | via ServiceLocator (hidden) | passed explicitly by Bootstrap |
| Files per feature | 2 (ViewModel + Controller) | 1 (Cubit) |
| Optimistic update | Controller reads ViewModel | Cubit reads its own `self.state` |

---

## ServiceLocator — where to use it

The ServiceLocator is a DI container. It has two distinct use cases in this architecture:

**Server** — Services and Repositories are registered in Bootstrap and resolved in Controllers.

```lua
-- In a Controller (server-side):
local dataSvc = ServiceLocator.Get("DataService")
```

**Client** — Cubits are registered in Bootstrap. Other Cubits that need to react to each other resolve via ServiceLocator internally.

```lua
-- In a Cubit that listens to another Cubit:
local invCubit = ServiceLocator.Get("InventoryCubit")
invCubit.OnStateChanged:Connect(function(state) ... end)
```

> **Views never use ServiceLocator.** Views receive their Cubit explicitly as a parameter from Bootstrap. This keeps Views pure and independently testable.

---

## Signal vs NetBridge

Both transport events, but they serve different purposes.

| | Signal | NetBridge |
|---|---|---|
| Scope | Same process (server↔server or client↔client) | Across the network (server↔client) |
| Replaces | BindableEvent | RemoteEvent / RemoteFunction |
| Type-safety | Yes | Yes (via consistent API) |
| When to use | Service notifying another Service, Cubit notifying a View | Server pushing data to client, client invoking server action |

```lua
-- Signal: in-process (e.g. DataService notifying Bootstrap)
dataSvc.OnCoinsChanged:Connect(function(player, newCoins)
    NetBridge.FireClient("CoinsUpdate", player, newCoins)
end)

-- NetBridge: cross-network (Bootstrap pushing to client)
NetBridge.FireClient("CoinsUpdate", player, newCoins)
```

---

## The Composition Root

Both Bootstrap files are **composition roots** — the only place where instances are created and wired together. Nothing else calls `.new()` on a Service, Repository, or Cubit. Dependencies are built in the correct order (low-level first) and injected downward.

```
Bootstrap.server.lua
  PlayerRepository.new()          ← no dependencies
  DataService.new(repo)           ← depends on repo
  InventoryService.new(dataSvc)   ← depends on DataService
  PlayerController.Init()         ← resolves from ServiceLocator
```
