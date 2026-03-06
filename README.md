# RblxArch — Clean Architecture for Roblox Studio

A structured, zero-boilerplate framework for Roblox games. Strict separation between server and client, reactive UI via the Cubit pattern, and dependency injection that makes your codebase actually maintainable.

**Zero Boilerplate** · **Client/Server Split** · **Cubit Pattern** · **Dependency Injection** · **DataStore Abstraction**

---

## Why this architecture?

A default Roblox project grows into a mess of spaghetti Scripts and LocalScripts with RemoteEvents scattered everywhere. This framework gives you a single, consistent answer to every question about where code lives and how it communicates.

| Problem | Solution |
|---|---|
| `game:GetService("Players").LocalPlayer` everywhere | `Context.GetPlayer()` — one line, anywhere |
| Services tightly coupled to DataStore | Repository pattern — swap the data source, zero changes elsewhere |
| ViewModel + Controller boilerplate per feature | Cubit — one class owns state and actions |
| RemoteEvents created all over the place | NetBridge — all network in one place |
| No clear answer to "where does this code go?" | Eight layers, each with a single defined responsibility |

## Data Flow

```
[Server]
  Repository  →  Service  →  Signal
                                └→  NetBridge.FireClient()

[Client]
  NetBridge.OnClient()  →  Cubit:method()  →  _emit(newState)
                                                    └→  OnStateChanged  →  View
```

## Documentation

| Page | Description |
|---|---|
| [Core Concepts](docs/concepts.md) | The eight layers and their responsibilities |
| [Folder Structure](docs/structure.md) | Where every file lives and why |
| [Framework Files](docs/framework.md) | The five core files you copy once |
| [Repository](docs/repository.md) | Abstracting the data source |
| [Service](docs/service.md) | Business logic layer |
| [Server Controller](docs/controller.md) | Handling client requests |
| [Cubit](docs/cubit.md) | Client state + actions |
| [View](docs/view.md) | Pure UI rendering |
| [Bootstrap](docs/bootstrap.md) | Wiring everything together |
| [Rules & Conventions](docs/rules.md) | Naming, structure, hard rules |

## Tutorial

| Step | Description |
|---|---|
| [1 — Setup](tutorial/01-setup.md) | Project structure and framework files |
| [2 — Types](tutorial/02-types.md) | Defining PlayerData |
| [3 — Repository](tutorial/03-repository.md) | Persisting data with DataStore |
| [4 — Service](tutorial/04-service.md) | Coins logic and player lifecycle |
| [5 — Server Bootstrap](tutorial/05-server-bootstrap.md) | Wiring the server |
| [6 — Cubit](tutorial/06-cubit.md) | Reactive coin state on the client |
| [7 — View](tutorial/07-view.md) | Binding the Cubit to the UI |
| [8 — Client Bootstrap](tutorial/08-client-bootstrap.md) | Wiring the client |
| [9 — Result](tutorial/09-result.md) | Running the game |
