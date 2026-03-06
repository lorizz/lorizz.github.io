# Framework Files

Five files. Copy them into `Shared/Framework/`. Never modify them after setup. They have no game-specific logic — only infrastructure.

---

## Signal.lua

Type-safe event emitter. Replaces `BindableEvent` entirely.

```lua
--!nonstrict

--- @class Signal
--- Type-safe event emitter. Replaces BindableEvent entirely.

local Signal = {}
Signal.__index = Signal

--- @return Signal
function Signal.new()
    return setmetatable({ _listeners = {} }, Signal)
end

--- Fires all listeners in separate threads.
--- @param ... any
function Signal:Fire(...)
    for _, cb in self._listeners do
        task.spawn(cb, ...)
    end
end

--- @param callback (...any) -> ()
--- @return Connection
function Signal:Connect(callback)
    table.insert(self._listeners, callback)
    return {
        Disconnect = function()
            local i = table.find(self._listeners, callback)
            if i then table.remove(self._listeners, i) end
        end
    }
end

--- Connects a callback that fires exactly once then disconnects itself.
--- @param callback (...any) -> ()
--- @return Connection
function Signal:Once(callback)
    local conn
    conn = self:Connect(function(...)
        conn:Disconnect()
        callback(...)
    end)
    return conn
end

--- Yields the current thread until the signal fires.
--- @return ...any
function Signal:Wait()
    local thread = coroutine.running()
    self:Once(function(...)
        task.spawn(thread, ...)
    end)
    return coroutine.yield()
end

--- Disconnects all active listeners.
function Signal:DisconnectAll()
    table.clear(self._listeners)
end

return Signal
```

---

## ServiceLocator.lua

Central DI container. Register in Bootstrap, resolve anywhere via `Get()`.

```lua
--!nonstrict

--- @class ServiceLocator
--- Central DI container. Register in Bootstrap, Get anywhere.

local ServiceLocator = {}
local _registry = {}
local _factories = {}

--- Registers a constructed instance under a name.
--- @param name string
--- @param instance any
function ServiceLocator.Register(name, instance)
    assert(_registry[name] == nil, ("Service '%s' is already registered"):format(name))
    _registry[name] = instance
end

--- Registers a factory called lazily on the first Get().
--- @param name string
--- @param factory () -> any
function ServiceLocator.RegisterFactory(name, factory)
    assert(_factories[name] == nil, ("Factory '%s' is already registered"):format(name))
    _factories[name] = factory
end

--- Retrieves a service. Resolves lazy factories on first call.
--- Throws if not registered.
--- @param name string
--- @return any
function ServiceLocator.Get(name)
    if _registry[name] == nil then
        local factory = _factories[name]
        assert(factory ~= nil, ("Service '%s' not found. Did you call Register()?"):format(name))
        _registry[name] = factory()
        _factories[name] = nil
    end
    return _registry[name]
end

return ServiceLocator
```

---

## Context.lua

Zero-boilerplate access to the most common client-side runtime values.

```lua
--!nonstrict

--- @class Context
--- Zero-boilerplate access to the most common client-side runtime values.

local RunService     = game:GetService("RunService")
local ServiceLocator = require(game.ReplicatedStorage.Shared.Framework.ServiceLocator)

local Context = {}

--- Returns LocalPlayer. Throws if called server-side.
--- @return Player
function Context.GetPlayer()
    assert(RunService:IsClient(), "GetPlayer() is only available on the client")
    return game.Players.LocalPlayer
end

--- Returns LocalPlayer's Character. Yields until it exists.
--- @return Model
function Context.GetCharacter()
    assert(RunService:IsClient(), "GetCharacter() is only available on the client")
    local player = game.Players.LocalPlayer
    return player.Character or player.CharacterAdded:Wait()
end

--- Retrieves a registered service from the ServiceLocator.
--- @param name string
--- @return any
function Context.GetService(name)
    return ServiceLocator.Get(name)
end

--- @return boolean
function Context.IsClient()
    return RunService:IsClient()
end

--- @return boolean
function Context.IsServer()
    return RunService:IsServer()
end

return Context
```

---

## Cubit.lua

Base class for all client Cubits. Subclass this — never instantiate directly.

```lua
--!nonstrict

--- @class Cubit
--- Base class for all client Cubits.
--- Holds state and notifies Views on every change via OnStateChanged.
--- Subclass this. Never instantiate directly.

local Signal = require(game.ReplicatedStorage.Shared.Framework.Signal)

local Cubit = {}
Cubit.__index = Cubit

--- @param initialState table
--- @return Cubit
function Cubit.new(initialState)
    return setmetatable({
        state          = initialState,
        OnStateChanged = Signal.new(),
    }, Cubit)
end

--- Replaces state and fires OnStateChanged.
--- Only call from within a Cubit subclass method.
--- @param newState table
function Cubit:_emit(newState)
    self.state = newState
    self.OnStateChanged:Fire(newState)
end

return Cubit
```

---

## NetBridge.lua

Centralized wrapper for all remote communication. Never create `RemoteEvent` or `RemoteFunction` manually.

```lua
--!nonstrict

--- @class NetBridge
--- Centralized wrapper for all remote communication.
--- Never create RemoteEvent or RemoteFunction manually.

local RunService        = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local NetBridge = {}

local function getFolder()
    if RunService:IsServer() then
        local f = ReplicatedStorage:FindFirstChild("_net") or Instance.new("Folder")
        f.Name   = "_net"
        f.Parent = ReplicatedStorage
        return f
    end
    return ReplicatedStorage:WaitForChild("_net")
end

--- [Server] Creates a RemoteEvent. Must be called in server Bootstrap before clients connect.
--- @param name string
function NetBridge.CreateEvent(name)
    assert(RunService:IsServer(), "CreateEvent() must be called server-side")
    local re  = Instance.new("RemoteEvent")
    re.Name   = name
    re.Parent = getFolder()
end

--- [Server] Creates a RemoteFunction with a handler.
--- @param name string
--- @param handler (player: Player, ...any) -> ...any
function NetBridge.OnServer(name, handler)
    assert(RunService:IsServer(), "OnServer() must be called server-side")
    local rf          = Instance.new("RemoteFunction")
    rf.Name           = name
    rf.Parent         = getFolder()
    rf.OnServerInvoke = handler
end

--- [Server] Fires a RemoteEvent to one specific client.
--- @param name string
--- @param player Player
--- @param ... any
function NetBridge.FireClient(name, player, ...)
    assert(RunService:IsServer(), "FireClient() must be called server-side")
    local re = getFolder():FindFirstChild(name)
    assert(re, ("RemoteEvent '%s' not found. Did you call CreateEvent()?"):format(name))
    re:FireClient(player, ...)
end

--- [Server] Fires a RemoteEvent to all connected clients.
--- @param name string
--- @param ... any
function NetBridge.FireAllClients(name, ...)
    assert(RunService:IsServer(), "FireAllClients() must be called server-side")
    local re = getFolder():FindFirstChild(name)
    assert(re, ("RemoteEvent '%s' not found. Did you call CreateEvent()?"):format(name))
    re:FireAllClients(...)
end

--- [Client] Subscribes to a RemoteEvent fired from the server.
--- @param name string
--- @param handler (...any) -> ()
--- @return Connection
function NetBridge.OnClient(name, handler)
    assert(RunService:IsClient(), "OnClient() must be called client-side")
    local re = getFolder():WaitForChild(name)
    return re.OnClientEvent:Connect(handler)
end

--- [Client] Invokes a RemoteFunction on the server and yields for the response.
--- @param name string
--- @param ... any
--- @return ...any
function NetBridge.InvokeServer(name, ...)
    assert(RunService:IsClient(), "InvokeServer() must be called client-side")
    local rf = getFolder():WaitForChild(name)
    return rf:InvokeServer(...)
end

return NetBridge
```
