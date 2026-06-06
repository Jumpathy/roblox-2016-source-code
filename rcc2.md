# Roblox Revival Backend — Architecture & Endpoint Reference

A reference for implementing a 2016-era Roblox web backend. This covers how the system works, what endpoints the client and RCCService actually require, the full SOAP envelope shapes for RCC communication, and minimum viable stub responses to get a game instance booting.

Architecture and storage decisions are left to you. The reference examples here use whatever language/framework you prefer — what matters is the HTTP contracts and SOAP structure, not the implementation.

---

## How It Works

The client (Studio / RobloxPlayerBeta) is pointed at `localhost` instead of `roblox.com`. Every HTTP call it would normally make to Roblox's servers lands on your backend instead. Your server is the entire Roblox API surface as far as the client is concerned.

**RCCService** runs the actual game instances and is a separate process you already have. Your backend orchestrates it over SOAP — it does not run game logic itself.

There are three moving parts:

- **Your web backend** — handles all the HTTP endpoints below
- **RCCService** — spins up and runs game instances on demand
- **A lease-renewal loop** — keeps active instances alive by periodically pinging RCC; instances have a 300s lease and are reaped if not renewed

For the renewal loop: since web requests are short-lived, you'll want a separate long-running process or cron job that tracks active jobs and fires `RenewLease` every ~60s. Store active job state (jobId → host/port) somewhere both the web layer and the renewal loop can see it (a database, Redis, SQLite, etc.).

---

## A Full Join, Start to Finish

1. Client GETs `/Game/PlaceLauncher.ashx?placeId=X`. You allocate a UDP port, mint a `jobId`, and send RCCService an `OpenJob` SOAP request. Respond with JSON including `status: 2` and the join/auth URLs.

2. RCCService loads `gameserver.txt` inside a sandboxed Lua instance and starts listening on the UDP port you gave it. That Lua calls `game:Load(baseUrl .. "/asset/?id=" .. placeId)` — so **`/asset/?id=<placeId>` must return the real `.rbxl` file** or the instance comes up empty.

3. Client GETs `/game/authenticate.ashx` to validate and rotate its ticket.

4. Client GETs `/game/join.ashx?jobId=&port=&placeId=` and receives the join configuration blob (a JSON object with a required leading `\n`).

5. Client opens a direct UDP connection to `MachineAddress:ServerPort` from the join blob — that's the actual game connection to RCCService.

6. Your renewal loop fires `RenewLease` every ~60s per active job. If the 300s lease lapses, RCCService kills the instance.

7. Throughout all of this, the client fires dozens of side-calls — analytics, version checks, feature flags, counters. These all need a `200` so the client doesn't stall. See the stub table at the bottom.

**Everything is dynamic per launch:** `jobId`, the instance UDP port, RCCService's own port if you're not on the default `64989`, tickets, session IDs. None of it is reusable between sessions.

---

## SOAP to RCCService

All RCC calls are `POST` to `http://<RCC_HOST>:<RCC_PORT>` (default `64989`, treat as configurable) with these headers:

```
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://roblox.com/<Action>"
```

Every request uses this envelope, with the action XML inside `<soap:Body>`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:ns1="http://roblox.com/">
  <soap:Body>
    <!-- action XML here -->
  </soap:Body>
</soap:Envelope>
```

Always XML-escape values going into the envelope (`&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`).

---

### OpenJob

Starts a game instance. The `script` body is your `gameserver.txt` contents with `start(...)` appended so it actually runs. The three `LuaValue` arguments are passed into that Lua as `...` — placeId, the UDP port you're assigning, and your base URL.

```xml
<ns1:OpenJob>
  <ns1:job>
    <ns1:id>job-1818-1749200000000</ns1:id>
    <ns1:expirationInSeconds>300</ns1:expirationInSeconds>
    <ns1:category>0</ns1:category>
    <ns1:cores>1</ns1:cores>
  </ns1:job>
  <ns1:script>
    <ns1:name>GameScript</ns1:name>
    <ns1:script>-- your gameserver.txt contents here, with "\nstart(...)\n" appended</ns1:script>
    <ns1:arguments>
      <ns1:LuaValue><ns1:type>LUA_TNUMBER</ns1:type><ns1:value>1818</ns1:value></ns1:LuaValue>
      <ns1:LuaValue><ns1:type>LUA_TNUMBER</ns1:type><ns1:value>2006</ns1:value></ns1:LuaValue>
      <ns1:LuaValue><ns1:type>LUA_TSTRING</ns1:type><ns1:value>http://localhost</ns1:value></ns1:LuaValue>
    </ns1:arguments>
  </ns1:script>
</ns1:OpenJob>
```

---

### RenewLease

Fire this every ~60s per active job. If the lease expires (300s), RCCService reaps the instance.

```xml
<ns1:RenewLease>
  <ns1:jobID>job-1818-1749200000000</ns1:jobID>
  <ns1:expirationInSeconds>300</ns1:expirationInSeconds>
</ns1:RenewLease>
```

---

### CloseJob

Call on shutdown or when a session ends.

```xml
<ns1:CloseJob>
  <ns1:jobID>job-1818-1749200000000</ns1:jobID>
</ns1:CloseJob>
```

---

### BatchJob

One-shot Lua execution, used for thumbnail generation. Send a Lua script that loads a place or asset and calls `ThumbnailGenerator:Click(...)` — RCCService runs it and returns the result. Response comes back in the SOAP reply as a `LuaValue`.

```xml
<ns1:BatchJob>
  <ns1:job>
    <ns1:id>thumb-1749200000000-abc1234</ns1:id>
    <ns1:expirationInSeconds>60</ns1:expirationInSeconds>
    <ns1:category>0</ns1:category>
    <ns1:cores>1</ns1:cores>
  </ns1:job>
  <ns1:script>
    <ns1:name>ThumbnailScript</ns1:name>
    <ns1:script>-- Lua that calls ThumbnailGenerator:Click() and returns the result</ns1:script>
    <ns1:arguments>
      <ns1:LuaValue><ns1:type>LUA_TNUMBER</ns1:type><ns1:value>1818</ns1:value></ns1:LuaValue>
      <ns1:LuaValue><ns1:type>LUA_TSTRING</ns1:type><ns1:value>http://localhost</ns1:value></ns1:LuaValue>
    </ns1:arguments>
  </ns1:script>
</ns1:BatchJob>
```

---

### LuaValue Types

When building `<arguments>`, the valid types are:

| Type | Value format |
|---|---|
| `LUA_TNIL` | empty `<value></value>` |
| `LUA_TBOOLEAN` | `true` or `false` |
| `LUA_TNUMBER` | numeric string |
| `LUA_TSTRING` | XML-escaped string |

---

### Parsing SOAP Responses

Check for a `<Fault>` element first — the `<faultstring>` or `<Text>` inside it is the error message.

For BatchJob results, the return value comes back as a `LuaValue` in the response. Pull the `<type>`/`<value>` pair — a `LUA_TSTRING` value is your base64-encoded image or string return. Responses come in two shapes depending on the RCC version:

- **Form 1:** explicit `<LuaValue>` wrappers containing `<type>` and `<value>` children (array responses)
- **Form 2:** bare `<type>` immediately followed by `<value>` (single-return BatchJob)

Handle both.

---

## Required HTTP Endpoints

These are what the client actually depends on. The client will stall or refuse to launch without them.

---

### `GET /Game/PlaceLauncher.ashx?placeId=`

Fire `OpenJob` to RCCService, then respond with JSON. `status: 2` means ready — the client polls otherwise.

```json
{
  "jobId": "job-1818-1749200000000",
  "status": 2,
  "joinScriptUrl": "http://localhost/game/join.ashx?jobId=job-1818-...&port=2006&placeId=1818",
  "authenticationUrl": "http://localhost/game/authenticate.ashx",
  "authenticationTicket": "ticket-job-1818-..."
}
```

---

### `GET /game/authenticate.ashx?suggest=` and `POST /game/authenticate.ashx`

The ticket handshake. The client suggests a ticket via `?suggest=`; you validate and return a rotated one as the plain-text response body. That returned string becomes the ticket the engine trusts going forward (`AuthenticationMarshallar::Authenticate` in the C++ client).

- If `suggest` starts with `tkt-`: validate it, mint a fresh `tkt-<random>` bound to the same user, return it as `text/plain`
- Anything else (legacy/Studio path): return `OK` as `text/plain` with `200`
- Invalid/unknown ticket: `401` with empty body
- `POST` to the same URL: always `200 OK`

For single-user local setups you can keep tickets fully deterministic (`ticket-<jobId>`) and skip the rotation logic entirely. Rotation only matters once you need to distinguish between multiple authenticated users.

---

### `GET /game/join.ashx?jobId=&port=&placeId=`

Returns the join configuration. **The response body must be prefixed with a literal `\n`** (the 2016 client's join-script parser requires it). `Content-Type: text/plain`.

```jsonc
{
  "ClientPort": 0,
  "MachineAddress": "localhost",
  "ServerPort": 2006,
  "PingUrl": "",
  "PingInterval": 20,
  "UserId": 1,
  "ScreenShotInfo": "",
  "VideoInfo": "",
  "CreatorId": 1,
  "CreatorTypeEnum": "User",
  "MembershipType": "None",        // None | BuildersClub | TurboBuildersClub | OutrageousBuildersClub
  "AccountAge": 365,
  "CookieStoreFirstTimePlayKey": "rbx_evt_ftp",
  "CookieStoreFiveMinutePlayKey": "rbx_evt_fmp",
  "CookieStoreEnabled": true,
  "IsRobloxPlace": true,
  "GenerateTeleportJoin": false,
  "IsUnknownOrUnder13": false,
  "SessionId": "session-job-1818-...",
  "CharacterAppearance": "http://localhost/Asset/CharacterFetch.ashx?userId=1",
  "ClientTicket": "ticket-job-1818-...",
  "GameId": "1818",                // placeId as string
  "PlaceId": 1818,
  "BaseUrl": "http://localhost",
  "ChatStyle": "ClassicAndBubble",
  "SuperSafeChat": false,
  "FollowUserId": 0,
  "UniverseId": 1818,
  "BrowserTrackerId": "0",
  "UserName": "Player"
}
```

---

### `GET /game/visit.ashx`

Studio's Play Solo join script. Studio `loadfile()`s this URL and executes the returned Lua. `Content-Type: text/plain`. This Lua configures service URLs (Social, MarketplaceService, etc.) to point back at your localhost endpoints, creates the local player, and starts the run loop.

---

### `GET /Game/LoadPlaceInfo.ashx`

Fetched by `gameserver.txt` Lua running inside the RCCService instance to configure service URLs. `Content-Type: text/plain`. Returns Lua — a set of `pcall(function() game:GetService(...):Set___Url(...) end)` calls pointing Social/GamePass/etc. at your localhost endpoints.

---

### Assets — not stubbable

**`GET /asset/?id=`**

Must serve real file data. Two cases:

- **`id` == a placeId**: return the raw `.rbxl` file. RCCService's `game:Load()` call fetches this. No file = empty dead instance with nothing loaded.
- **`id` == an asset ID**: return the raw asset bytes (mesh, texture, clothing, etc.). No assets = no characters, no meshes rendering.

Validate that `id` is numeric. `400` on missing/invalid, `404` on not found. This is a real asset server — serve from disk, or proxy and cache from an upstream source.

**`GET /Asset/CharacterFetch.ashx?userId=`**

Returns a semicolon-delimited list of asset URLs that make up the avatar. `Content-Type: text/plain`.

```
http://localhost/Asset/CharacterBodyColors.ashx?userId=1;http://localhost/Asset/?id=63690008;http://localhost/Asset/?id=144076358
```

First entry is always the body colors URL; the rest are worn asset IDs.

**`GET /Asset/CharacterBodyColors.ashx?userId=`**

Returns a Roblox XML `BodyColors` item with the six BrickColor int IDs. `Content-Type: text/xml`.

```xml
<roblox xmlns:xmime="http://www.w3.org/2005/05/xmlmime"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="http://www.roblox.com/roblox.xsd"
        version="4">
  <Item class="BodyColors">
    <Properties>
      <string name="Name">Body Colors</string>
      <int name="HeadColor">9</int>
      <int name="LeftArmColor">9</int>
      <int name="LeftLegColor">9</int>
      <int name="RightArmColor">9</int>
      <int name="RightLegColor">9</int>
      <int name="TorsoColor">9</int>
    </Properties>
  </Item>
</roblox>
```

BrickColor `9` is white. Replace with values from your user's avatar data.

**`POST /ide/publish/UploadExistingAsset?assetId=`**

Studio publishes a place via POST. Store the request body bytes so `/asset/?id=<assetId>` can serve them back. Return the assetId as `text/plain`.

---

### Auth / Machine Validation

Studio refuses to start without these.

| Method | Path | Response | Type | Notes |
|---|---|---|---|---|
| GET | `/game/GetCurrentUser.ashx` | `1` | text/plain | Must be a positive integer string (the userId) |
| GET | `/login/RequestAuth.ashx` | `http://localhost/login/negotiatewithcookie` | text/plain | Returning localhost triggers the webkit auth-skip; avoids a ~10s hang |
| GET | `/login/negotiatewithcookie` | `OK` | text/plain | `200` = authenticated |
| POST | `/game/validate-machine` | `{"success":true}` | json | POSTs MAC addresses; **must** return `success: true` or Studio won't start |
| GET | `/universes/validate-place-join` | `true` | text/plain | Checked before join is allowed |
| GET | `/game/logout.aspx` | `OK` | text/plain | Sign-out |

---

### Feature Flags

**`GET /Setting/QuietGet/ClientAppSettings/:group`** and **`GET /Setting/QuietGet/:group`**

Return a JSON object of feature flags: `{ "FlagName": value }`. An empty `{}` is enough to boot. Specific flags control client behavior you can tune later.

---

## Stub Endpoints

These all return `200` with a benign body. They keep the client from stalling but don't implement real functionality — **DataStore stubs mean data is silently lost, GamePass checks always return false, analytics vanish.** Good enough to get a game running; replace with real implementations when the feature matters.

| Path(s) | Method(s) | Response | Type |
|---|---|---|---|
| `/GetAllowedSecurityVersions`, `/GetAllowedSecurityKeys`, `/GetAllowedMD5Hashes`, `/GetAllowedMemHashes` | GET | `{"data":[]}` | json |
| `/GetCurrentClientVersionUpload` | GET | `{"version":"0.0.0.0","clientVersionUpload":""}` | json |
| `/Error/Grid.ashx`, `/Error/Dmp.ashx`, `/Error/Breakpad.ashx` | GET, POST | `OK` | text/plain |
| `/Analytics/LogFile.ashx`, `/Analytics/Measurement.ashx`, `/Analytics/ContentProvider.ashx` | GET, POST | empty | text/plain |
| `/v1.1/Counters/Increment`, `/v1.0/MultiIncrement` | GET, POST | `OK` | text/plain |
| `/UploadMedia/PostImage.aspx`, `/UploadMedia/UploadVideo.aspx` | GET, POST | `OK` | text/plain |
| `/Game/JoinRate.ashx`, `/game/placevisit.ashx`, `/game/clientpresence.ashx`, `/game/cdn.ashx`, `/game/keepalivepinger.ashx`, `/game/report-stats` | GET | empty | text/plain |
| `/Game/MachineConfiguration.ashx` | POST | `{"ok":true}` | json |
| `/gametransactions/getpendingtransactions/` | GET | `[]` | text/plain |
| `/points/get-awardable-points` | GET | `{"points":"0"}` | json |
| `/game/gamepass/gamepasshandler.ashx` | GET | `<Value Type="boolean">false</Value>` | text/xml |
| `/game/badge/isbadgedisabled.ashx` | GET | `0` | text/plain |
| `/asset/getscriptstate.ashx` | GET | `0 0 0 0` | text/plain |
| `/persistence/getv2` | GET | `{"data":[]}` | json |
| `/persistence/getSortedValues` | GET | `{"data":{"Entries":[],"ExclusiveStartKey":null}}` | json |
| `/persistence/increment`, `/persistence/set` | GET, POST | `{"data":null}` | json |
| `/login/negotiate.ashx` | GET | empty | text/plain |

---

## The Discovery Loop

Don't try to enumerate every endpoint upfront. Add a catch-all that returns `200` + empty body and **logs the full unmatched URL with its query string**. Boot the client, watch the log, and each unhandled line tells you the next thing to implement. Search the client binary's strings for the path fragment to find the expected response shape, then either stub it from the table above or build it for real.

Return `200`/empty from the catch-all rather than `404` — it keeps the client moving so you can collect several missing routes per boot instead of dying on the first one.

A few endpoints gate startup entirely and need to be correct before anything else works: `validate-machine` (`success: true`), `GetCurrentUser.ashx` (positive int), `validate-place-join` (`true`), and `RequestAuth.ashx` (the localhost redirect). If the client won't launch at all, start there.
