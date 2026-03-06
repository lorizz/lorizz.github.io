# Rules & Conventions

---

## Naming

| Thing | Convention | Example |
|---|---|---|
| Repository | PascalCase + `Repository` suffix | `PlayerRepository`, `LeaderboardRepository` |
| Service | PascalCase + `Service` suffix | `DataService`, `ShopService`, `MatchService` |
| Controller | PascalCase + `Controller` suffix | `PlayerController`, `ShopController` |
| Cubit | PascalCase + `Cubit` suffix | `HUDCubit`, `InventoryCubit`, `ShopCubit` |
| View | PascalCase + `View` suffix | `HUDView`, `InventoryView`, `ShopView` |
| Signal fields | `On` + PascalCase event name | `OnCoinsChanged`, `OnItemAdded`, `OnMatchStart` |
| NetBridge events | PascalCase verb + noun | `CoinsUpdate`, `InventoryUpdate`, `MatchStart` |
| NetBridge functions | PascalCase verb + noun | `UseOrb`, `PurchaseItem`, `SpawnCharacter` |
| Type files | PascalCase + `Types` suffix | `PlayerTypes`, `GameTypes`, `ItemTypes` |

---

## Hard Rules

### General

**`--!nostrict` everywhere.** Luau strict mode produces false positives with metatables. Use `--!nostrict` in every file.

**The only Scripts in the project are the two Bootstrap files.** Every other file is a `ModuleScript`.

**Never use `require` at the top level of a Bootstrap file for a module that might fail.** Use `pcall` or ensure all dependencies exist before the Bootstrap runs.

---

### Server

**Repositories are server-only.** Never `require` a Repository from a LocalScript, a Cubit, or a View. They may call DataStore, HttpService, or other server-only APIs.

**Services never call NetBridge.** A Service fires a Signal. Bootstrap listens to that Signal and calls `NetBridge.FireClient`. Services have no knowledge of the network layer.

**Services never call other Services via ServiceLocator.** Dependencies between Services are resolved at construction time in Bootstrap and injected via the constructor.

**Always include `game:BindToClose`.** Without it, data is lost when the server shuts down in Studio or during a normal shutdown.

**All RemoteEvents and RemoteFunctions are created in `Bootstrap.server.lua`.** Never create them at runtime or inside a Service/Controller. They must exist before any client tries to use them.

---

### Client

**Views never call NetBridge.** Views call Cubit methods. Cubits call NetBridge.

**Views never use ServiceLocator.** Cubits are passed to Views explicitly by Bootstrap.

**Views never contain game logic.** A View reads `state.someValue` and renders it. Conditions about game rules belong in a Cubit or Service.

**Never mutate `self.state` directly in a Cubit.** Always call `self:_emit(newState)`. Direct mutation skips the Signal and leaves Views out of sync.

**Cubits that depend on other Cubits use ServiceLocator internally.** Never pass Cubits to each other via the constructor — that creates coupling in Bootstrap.

---

## Checklist for a new feature

Use this every time you add something new to the game.

```
Server:
  [ ] Define the data shape in Shared/Types/ if needed
  [ ] Create or update a Repository if new persistence is needed
  [ ] Create a Service with the business logic
  [ ] Register the Service in Bootstrap.server.lua
  [ ] Create a Controller with NetBridge.OnServer handlers
  [ ] Call Controller.Init() in Bootstrap.server.lua
  [ ] Create NetBridge events with CreateEvent() if server needs to push
  [ ] Wire Service Signals to NetBridge.FireClient in Bootstrap

Client:
  [ ] Create a Cubit with the state shape and actions
  [ ] Register the Cubit in Bootstrap.client.lua
  [ ] Create a View with Mount(gui, cubit)
  [ ] Call View.Mount in Bootstrap.client.lua
  [ ] Connect NetBridge.OnClient to Cubit methods in Bootstrap
```

---

## Where does this code go?

| Question | Answer |
|---|---|
| Code that reads or writes a data source | Repository |
| Code that validates a purchase or computes rewards | Service |
| Code that receives a client request and validates input | Server Controller |
| Code that holds what the UI should currently show | Cubit |
| Code that updates a TextLabel or Frame | View |
| Code that receives a RemoteEvent from the server | Bootstrap.client.lua → calls Cubit method |
| Code that fires a RemoteEvent to a client | Bootstrap.server.lua → Signal handler → NetBridge.FireClient |
| Code that constructs any class | Bootstrap (server or client) |
