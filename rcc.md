# How RCCService Starts and How Clients Connect to It

---

## How RCCService starts up

When you run it with `-Console` (the flag that makes it run in your terminal instead of
registering as a Windows service), this is what happens:

1. Binds a **SOAP/HTTP server on port 64989** (the default — you can override it by passing a
   plain number as an argument, e.g. `RCCService.exe -Console 64989`).
2. Sits in a loop calling `stepRCC()`, which calls `service.accept()` — just waiting for SOAP
   requests to come in over HTTP.
3. Each incoming SOAP request gets dispatched to a thread pool worker.

It does **not** run any game by itself just from starting. The game engine inside it is idle
until something tells it to open a "Job."

### The `-PlaceId:X` shortcut

There is a convenience path (`RCCService.cpp:801-847`) — if you pass `-PlaceId:123` on the
command line, RCCService will immediately call `OpenJob` on itself with a script that reads
`gameserver.txt` and appends:

```
start(123, 53640, 'http://www.roblox.com')
```

That boots the game engine, loads the place, and starts `NetworkServer` on port **53640** — all
from within itself, without needing an external orchestrator. This is the shortcut the devs used
for testing.

### What OpenJob (the SOAP call) actually does

When `OpenJob` is called (either by the `-PlaceId` shortcut or by the BackendServer), it:

1. Creates a new DataModel — a full game engine instance.
2. Executes the Lua script you passed in — which is `gameserver.txt` + `start(placeId, port, url)`.
3. `gameserver.txt`'s `start()` function configures all the services, loads the place via
   `game:Load(url .. "/asset/?id=" .. placeId)`, then calls `ns:Start(port)` — which starts
   RakNet UDP listening on whatever port was passed in.

---

## How WindowsClient connects to it

The client never connects directly to RCCService's SOAP port (64989). Instead it goes through a
web server (the BackendServer) and a Lua "join script" mechanism:

```
WindowsClient
  -> HTTP GET /Game/PlaceLauncher.ashx?placeId=X   (to the BackendServer)
  <- { status: 2, joinScriptUrl: "...", authenticationTicket: "..." }

  -> HTTP GET /game/join.ashx?port=2006&placeId=X  (fetches a Lua script)
  <- Lua script text

  -> Executes that Lua script inside the engine
       -> NetworkClient:PlayerConnect(userId, "localhost", 2006, 0, 30)
       -> RakNet UDP handshake -> RCCService's NetworkServer on that port
```

The join script is **a Lua script** the engine runs, not a config file. It sets up all the
service URLs and then calls `NetworkClient:PlayerConnect(userId, machineAddress, serverPort,
clientPort, timeout)`. That is the actual UDP connection to the game server
(`Network/GameConfigurer.cpp:667`).

---

## The BackendServer is the glue between them

`BackendServer/server.js` already implements this entire flow:

- **`/Game/PlaceLauncher.ashx`** — calls RCCService's SOAP `OpenJob`, allocates a game port
  starting at 2006, returns the join script URL to the client.
- **`/game/join.ashx`** — generates the Lua join script on the fly with the correct `localhost`
  address and port baked in.
- Keeps the RCC job alive by calling `RenewLease` every 60 seconds.
- RCCService address and port are configurable via `RCC_HOST` / `RCC_PORT` environment variables
  (defaults to `127.0.0.1:64989`).

---

## Full local stack to get a client connecting

```
Step 1 — Start RCCService
  RCCService.exe -Console
  SOAP server listens on :64989, game engine is idle.

Step 2 — Start the BackendServer  (run as Administrator — needs port 80)
  cd BackendServer
  node server.js

  Also add the hosts file entries listed at the top of server.js:
    C:\Windows\System32\drivers\etc\hosts  (open as Administrator)
    127.0.0.1   www.roblox.com
    127.0.0.1   assetgame.roblox.com
    127.0.0.1   clientsettings.api.roblox.com
    127.0.0.1   versioncompatibility.api.roblox.com
    127.0.0.1   ephemeralcounters.api.roblox.com
    127.0.0.1   api.roblox.com
    127.0.0.1   data.roblox.com
    127.0.0.1   gamepersistence.roblox.com

Step 3 — Launch WindowsClient
  -> hits /Game/PlaceLauncher.ashx on the BackendServer
  -> BackendServer sends OpenJob to RCCService via SOAP
  -> RCCService boots a DataModel, loads the place, starts NetworkServer on UDP :2006
  -> BackendServer returns the joinScriptUrl to the client
  -> client fetches the join script, executes it
  -> PlayerConnect -> UDP :2006 -> you're in the game
```

### Note on place loading

Inside `gameserver.txt`, `start()` calls:

```lua
game:Load(url .. "/asset/?id=" .. placeId)
```

If the placeId doesn't resolve to a real place file on the backend, the load fails silently and
the server starts with an empty world. To serve a real place, host the `.rbxl` file contents
behind `/Asset/?id=<placeId>` on the BackendServer, or just accept the empty baseplate and build
from there.
