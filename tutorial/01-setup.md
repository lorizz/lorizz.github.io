# Tutorial — Step 1: Setup

In this tutorial you will build a complete coin game from scratch using this architecture. A player joins, their coins are loaded from DataStore, they earn one coin per second, and the balance is shown live in a UI label.

**What you will build:**

```
Player joins → data loads → coins shown in UI
Every second → +1 coin → UI updates live
Player leaves → data saves
Player rejoins → coins persist from last session
```

---

## Prerequisites

- Roblox Studio installed
- Basic knowledge of Luau and Studio's Explorer panel
- No prior framework knowledge required

---

## Project structure

Before writing any code, set up the folder structure. Create the following instances in the Explorer panel. Every file marked as `ModuleScript` is a `ModuleScript`. The two Bootstrap files are `Script` and `LocalScript` respectively.

```
ReplicatedStorage/
└── Shared/                          [Folder]
    ├── Framework/                   [Folder]
    │   ├── Signal                   [ModuleScript]
    │   ├── ServiceLocator           [ModuleScript]
    │   ├── Context                  [ModuleScript]
    │   ├── Cubit                    [ModuleScript]
    │   └── NetBridge                [ModuleScript]
    └── Types/                       [Folder]
        └── PlayerTypes              [ModuleScript]

ServerScriptService/
└── Server/                          [Folder]
    ├── Bootstrap                    [Script]          ← Script, not ModuleScript
    ├── Repositories/                [Folder]
    │   └── PlayerRepository         [ModuleScript]
    ├── Services/                    [Folder]
    │   └── DataService              [ModuleScript]
    └── Controllers/                 [Folder]
        └── PlayerController         [ModuleScript]

StarterPlayerScripts/
└── Client/                          [Folder]
    ├── Bootstrap                    [LocalScript]     ← LocalScript, not ModuleScript
    ├── Cubits/                      [Folder]
    │   └── HUDCubit                 [ModuleScript]
    └── Views/                       [Folder]
        └── HUDView                  [ModuleScript]
```

> Roblox appends `.server` and `.client` to Script and LocalScript names in some editors. In Studio, just name them `Bootstrap` and make sure their instance type is correct.

---

## Framework files

Copy the five framework files verbatim. You can find the full source in [Framework Files](../docs/framework.md).

- `Shared/Framework/Signal`
- `Shared/Framework/ServiceLocator`
- `Shared/Framework/Context`
- `Shared/Framework/Cubit`
- `Shared/Framework/NetBridge`

Once copied, do not modify them.

---

## GUI setup

Create the following GUI hierarchy in `StarterGui`:

```
StarterGui/
└── MainGui                          [ScreenGui]
    └── HUD                          [Frame]
        └── CoinsLabel               [TextLabel]
```

Position and style `CoinsLabel` however you like. The framework only cares that the path `MainGui > HUD > CoinsLabel` exists.

---

Next: [Step 2 — Types](02-types.md)
