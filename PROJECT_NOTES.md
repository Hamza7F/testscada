# PROJECT NOTES — scada_demo (myscada1-n0)

**Analysis date:** 2026-10-06
**Method:** every source file read in full by 5 parallel agents (Domain+Application,
Infrastructure, API, Web, MAUI+tests+mobile+root). This file is the compressed,
verified index. Where AGENTS.md disagrees with the code, **the code wins** and the
disagreement is called out.

---

## 1. REPO SHAPE

```
C:\Users\shan\myscada1-n0
├── scada_demo_test.slnx        7 projects (XML solution format)
├── AGENTS.md                   205 KB handover log (partly stale, see §9)
├── DB_LOCAL_SWITCH_CHANGELOG.md
├── README.md                   STALE + contains a WiFi password
├── README-solution.txt         references a non-existent .sln
├── package.json                AI Studio / Linux container runner (NOT mobile)
├── dotnet-tools.json            dotnet-ef 10.0.0, rollForward:false
├── dy                          stray unrelated CoreUI React bundle — delete
├── scada_db_iiot.sqlite(+wal/shm)  COMMITTED sqlite DB
├── backup_postgres_migrations/ 3 files, cannot compile (Npgsql commented out)
├── mosquitto/config/           orphan 8-line config, zero references
├── mobile/                     abandoned Expo app, pure mock
├── tests/ScadaEngine.Tests/    xUnit, 14 test methods, REAL
├── firmware/                   ***MISSING*** (documented everywhere, gone)
└── src/
    ├── scada_demo_test.Domain          (43 files)
    ├── scada_demo_test.Application     (17 files)
    ├── scada_demo_test.Infrastructure  (25 files)
    ├── scada_demo_test.API             (17 files)
    ├── scada_demo_test.Web             (21 files + wwwroot)
    └── scada_demo_test.Maui            (net9.0 — INCONSISTENT)
```

191 .cs/.razor source files in `src/`. Everything is `net10.0` **except MAUI (`net9.0`)**.

---

## 2. ARCHITECTURE (verified)

```
API (Program.cs = entrypoint, binds everything)
 ├── Infrastructure ──► Application ──► Domain
 └── Application
Web (Blazor Server) ──► Domain (constants only); everything else over HTTP
Maui ──► Domain only (Firebase, never the API)
```

* **DB provider is chosen at startup by OS** (`API/Program.cs:30-37`):
  Windows → `UseSqlServer` LocalDB `scada_db_iiot` + `MigrateAsync` + T-SQL schema init.
  Non-Windows → `UseSqlite("Data Source=scada_db_iiot.sqlite")` + `EnsureCreatedAsync`.
  Evaluated **once at boot** — flipping OS requires a restart.
* **Telemetry store is per-sensor dynamic tables** `telemetry_sensor_{type}_{id8}`,
  created by `SensorTelemetryRepository`. Columns come only from the driver catalog.
* **Two ingestion paths:** Modbus TCP poll (USR-W610 gateways) and HTTP push
  (Norvi ESP32, `/api/telemetry/tank-reading`).

---

## 3. THE 4 DRIVERS (all verified in code)

| Key | Class | FC | Identify window | Columns | Proof window | Default slave/poll |
|---|---|---|---|---|---|---|
| `AOSONG_AQ3485` | AosongAQ3485Driver | 03 | @0 qty2 (4 bytes) | TemperatureC / HumidityRH | **none** | 1 / 5 s |
| `KAIFENG_EM_FLOWMETER` | ElectromagneticFlowmeterDriver | 03 | @90 qty2 | InstantaneousFlowRate / AccumulatedTotalizer | @92 qty2 | 2 / 3 s |
| `VORTEX_FLOWMETER` | VortexFlowmeterDriver | 04 | @1026 qty2 | same flow columns | @1067 qty2 | 1 / 3 s |
| `SELEC_POWER_METER` | SelecPowerMeterDriver | 04 | @42 qty2 | same columns (kW/kWh) | @64 qty2 | 5 / 5 s |

* **There is NO `KAIFENG_FLOWMETER` (thermal mass) driver in the source**, even though
  AGENTS.md, `Gateway.razor` ParamCatalog, `AppTabs` and comments all refer to it.
* `ModbusValueCodec` decodes IEEE-754 float32 **high-word-first** (Aosong/EM/Vortex) and
  **low-word-first** (Selec), and folds exact `-0.0` → `+0.0`.
* Vortex reg 1026 is a **percentage**; driver converts `pct/100 * FullScaleFlowM3H(1000)`.
* **Startup log line `Modbus IIoT polling worker started (drivers: …)` is the ground truth**
  for which drivers are actually polled. If a driver is in the catalog but not in DI, it
  is never read.

---

## 4. SCADA ENGINE — the identity pipeline (Domain/ScadaEngine + Infrastructure/ScadaEngine)

Namespace is `ScadaEngine.Core.*` (inconsistent with `scada_demo_test.*` everywhere else).

**Envelope**: `ModbusDevicePacket` — immutable, 5 constructor guards (slave 1..247,
profile != Unknown, non-blank key/name, non-empty payload), payload cloned on read,
`ConfidenceScore` is hard-coded `Verified`.

**Fingerprint**: `CheckpostFingerprintEngine.Inspect(RawScanResponse)`
→ strict **uniqueness** (`matches.Length != 1` ⇒ reject, no ranking)
→ hint may reject a mismatch but can never supply missing evidence
→ every driver window needs ≥1 sample with exact byte length
→ proof window (if any) must pass `ValidateCorroboration`; a driver with **no** proof
  window needs **≥2 samples** of window[0].

**Dispatch**: `SensorDriverDispatcher` — FrozenDictionary profile → calculator
(`AQ3485Ghar`, `V880BRGhar`, `KaifengThermalGhar` **absent**, `ElectromagneticGhar`,
`SelecPowerGhar`). 9-condition rejection chain. Note `ElectromagneticGhar` is the only
one that decodes manually instead of calling `ParseData`.

**3-bucket Dual Guard** (`ModbusScannerService.ExecuteDualGuardPipeline`):
1. **Bucket 1 "Already In System"** — DB-registered slave ids, never probed.
2. **Bucket 2 "Blocked Scan Duplicates"** — `GroupBy(SlaveId).Count() > 1` ⇒ blocked,
   never reaches the Found Box, message names the colliding meters.
3. **Bucket 3 "Unique Found Box"** — single response ⇒ tagged + dispatched.

---

## 5. MODBUS SCANNER — real current algorithm

* **One tier only.** `tiers = [ (start, end, Clamp(passes,1,3)) ]`. The API default
  `passes=0` silently becomes **1**. `QuickScanEndAddress`, `ScanPassesQuick`,
  `ScanPassesFull`, `ConflictVoteThreshold`, all `ForeignCollision*` constants are
  **DEAD** (never read).
* Per address: Guard 1 skip if registered → probe every driver → build evidence
  (all own windows + proof retried up to `DegeneracyProofAttempts=8`, or a 2nd read of
  window[0] when there is no proof) → `CheckpostFingerprintEngine.Matches` gate.
* **`CandidateRank` currently degenerates**: `Unproven/Live/Extra` are all hard-coded 0
  and `Width` compares **DESC**, so the effective order is *widest window first, then
  catalog order*. All 4 drivers use qty=2 ⇒ width ties ⇒ **Aosong always outranks the
  rest**. This is the inverse of the documented "narrowest = most specific" rule.
* **Ambiguity is dead**: `ambiguityMap` is created, never populated ⇒
  `IsAmbiguous=false`, `Candidates=null` on every result.
* Per-address isolation; gateway declared unreachable only after
  `UnreachableFailureStreak = 12` consecutive failures, and that **clears all state**.
* `ModbusScanCoordinator` gives the poller and the scanner a **shared per-host:port
  semaphore** so a scan and a poll never share the socket.

---

## 6. API SURFACE (verified route → policy)

| Route | Policy |
|---|---|
| `GET /api/devices`, `GET /api/devices/{id}` | **NONE** ⚠ |
| `POST /api/devices/{id}/scan`, `GET …/reachability` | `Action:Devices.View` |
| `POST /api/devices`, `PUT`, `DELETE` | `Action:Devices.Add` / `.Edit` / `.Delete` |
| `GET /api/sensors/libraries` | **NONE** ⚠ |
| `GET /api/sensors`, `…/telemetry`, `export-csv`, `export-pdf` | `Action:Devices.View` |
| `POST /api/sensors` | `Action:Devices.Add` |
| `PUT /api/sensors/{id}` | `Action:Devices.Add` ⚠ (should be Edit) |
| `DELETE /api/sensors/{id}` | `Action:Devices.Delete` |
| `POST /api/telemetry/tank-reading` | **anonymous** (by design) |
| `POST /api/auth/login|refresh|logout`, `GET /api/auth/me` | anonymous / `[Authorize]` |
| alerts (6), audit-logs, sites, tanks, users (4), roles (6) | mixed — **users/roles have NO `[Authorize]` at all** ⚠ |

* 32 policies auto-built: `Tab:{key}` (14) + `Action:{key}` (18).
  Assertion = `IsSuperAdmin` claim OR `IsInRole("SuperAdmin")` OR matching `perm` claim.
  **Policy name is always `"Action:" + AppPermissions.X`** — never the bare dotted string.
* Scan **always returns HTTP 200** (no 502 path exists any more); unreachability is
  reported inside the payload as `connectivityOk=false` + `errorCode`.
* `/api/health` **does not exist** — the entire communication-health feature
  (item 38) is absent from source: no controller, no tracker, no DTOs, no UI.
* Guards that DO exist: duplicate-IP (409), device capacity 25 (400), slave-address
  uniqueness per device (400), driver-must-exist (400), ≥1 measurable parameter (400).

---

## 7. WEB (Blazor Server, :5150)

* Auth = cookie → forwards JWT as Bearer via `PermissionForwardingHandler`.
  `BuildApiClient` builds a **fresh handler per client with an explicit `InnerHandler`** —
  sharing a `DelegatingHandler` or leaving `InnerHandler` null is what previously broke
  login entirely. Two clients: 30 s normal, 5 min for the bus scan.
* `POST /account/login` is a minimal-API endpoint in `Web/Program.cs`, **not** a Blazor
  route. `GET /account/login` is a 404.
* `UseStaticWebAssets()` is guarded by a manifest check so `/_framework/blazor.server.js`
  resolves even in Production (the "page looks perfect but nothing clicks" bug).
* 12 pages, all real-data. Nav: Dashboard `/dashboard`, Live `/`, Summary `/summary`,
  TREND group (Plant Energy `/energy`, Charts `/charts`, Tanks `/tanks`),
  Devices group (Norvi `/devices`, Gateway `/gateway`), Reports, Alerts, Multi-Site,
  Audit Logs, Users.
* 5 scan DTOs mirror the server 3-bucket contract, incl. `AlreadyInSystemSlaves` and
  `DuplicateIdConflictDto.CollidingMeterNames`. Gateway.razor renders all three buckets.
* `LiveTelemetryState` subscribes SignalR `ReceiveReading` + `ReceiveAlertIncident` only.

---

## 8. WHAT IS STILL MOCK / DEAD / RISKY

**Main stack (API + Web + Infrastructure): clean of simulation** — verified, only
`ModbusPollingHostedService` and `RollupCompressionHostedService` are registered.

| Issue | Where |
|---|---|
| MAUI boots in **simulation** (`IsSimulationMode` defaults true) and fabricates Modbus hex every tick; **never calls the API** | `Maui/Services/*` |
| `mobile/` Expo app: 100% mock, **zero HTTP layer**, uses the deleted `norvi-slave-{i}` id scheme | `mobile/` |
| `firmware/` folder **missing** though documented in AGENTS.md + README | repo root |
| Firebase still hard-coded and still used by Web (legacy users/roles/tanks/sites/firmware) and all of MAUI | `Domain/Services/FirebaseScadaService.cs` |
| `UsersController` + `RolesController` have **no authorization** | API |
| Committed secrets: WiFi password (README), Firebase key, Gmail app password, Supabase string, SuperAdmin password; login page **pre-fills** the admin password | multiple |
| `.gitignore` is a Next.js template — ignores none of `bin/ obj/ .vs/ *.sqlite*` | root |
| sqlite DB + WAL committed | root |
| Git: **clone only, no local commits** (AGENTS.md claims a local `git init` d31861f) | `.git` |
| Dead code: `RequireUsersTabAttribute`, `TelemetryIngestService`, `TelemetryBroadcastBus` (never published), `FlowMeterCard.razor`, `charts.js drawComparisonChart`, Firmware OTA block (client+CSS), MAUI `LogsPage`, `dy`, `mosquitto/` | multiple |

**Web-side bugs worth fixing:**
1. `Gateway.razor` sends **non-ASCII unit strings** (`°C`, `m³`) in `SelectedParameters`
   → historically causes HTTP 400 on sensor create.
2. `&mdash;` written inside C# initializers in `Energy/PlantEnergy/Tanks.razor`
   → renders as literal text.
3. `Charts.razor` comparison tab ignores the selected time window.
4. `Index.razor` opens a bearer-protected PDF URL via `window.open` → 401.
5. `Devices.razor` children render only inside `@if (expanded)` (Gateway does it right).
6. `Login.razor` ships pre-filled admin credentials.
7. `ScanGatewayBusAsync` ends in a bare `catch { return null; }` → every scan failure
   shows the same "gateway didn't respond" message.
8. `ScadaDemoTestApiClient` still fabricates data in `GetHistoryAsync` (×0.95/×1.05) and
   the `CreateTankAsync` fallback.
9. `UseExceptionHandler("/Error")` points at a page that does not exist.

---

## 9. AGENTS.md vs ACTUAL SOURCE — read this before trusting AGENTS.md

| AGENTS.md claim | Reality in code |
|---|---|
| 5 drivers, `KAIFENG_FLOWMETER` thermal-mass driver | **4 drivers**, no thermal-mass driver |
| `/api/health` + `QualityBadge` + `WhatsThisTooltip` (item 38, "verified live") | **none of it exists** |
| Scan reachability pre-check, progress timer, honest estimate (items 33-35, 44-45) | **absent** from `Gateway.razor` |
| Foreign-window collision detection, vote gate (items 40-45) | constants + method are **dead code** |
| Ambiguity flagging + operator confirm (item 37) | `ambiguityMap` never populated |
| Selec exposes 61 parameters (item 37) | 2 columns only |
| `Energy.razor` has `/energy` alias + 2000 ms refresh (items 23, 28) | `/dashboard` only, 4000 ms |
| `NavMenu.OpenGroups` defaults to TREND+Devices | initialised empty |
| `Devices.razor` always-visible children (item 36) | behind `@if (expanded)` |
| Alerts "ALL" moved to bottom + metric gating (item 26) | "ALL" is first, no gating |
| Named `scada-bus-scan` HttpClient via IHttpClientFactory (item 33) | static `BuildApiClient` helper |
| Deleting a sensor keeps its telemetry table | **the table is dropped** |
| Push unknown-device log lowered to Debug (item 27) | still `LogWarning` |
| Item-45 bus map 1=Aosong/2=Vortex/3=EM/4=Selec | superseded by the 2026-10-01 measured map 1=EM/2=Vortex/3=Aosong/4=Selec |

---

## 10. TESTS

`tests/ScadaEngine.Tests` — xUnit 2.9.3, 2 classes, **14 test methods / ~20 cases**, all
green, referencing Infrastructure only. Uses a loopback `FakeGateway` Modbus TCP double.
They cover exactly the fragile parts: exception-code parsing, framing rejection, timeout
session invalidation, registered-id skipping, duplicate blocking (including identical
profiles), hint/byte-length insufficiency, genuine zero values, packet immutability.

**There are no tests for the API, Web, Domain entities, or the poller.**

---

## 11. RUN / BUILD

```
API : dotnet run --project src/scada_demo_test.API      (launchSettings: 0.0.0.0:5080)
Web : dotnet run --project src/scada_demo_test.Web      (localhost:5150)
Build only the changed project — never the whole .slnx (MAUI is slow / net9.0).
Login: superadmin@alamiot.com / SuperAdmin@123
Health check: curl -o NUL -w "%{http_code}" http://localhost:5150/_framework/blazor.server.js  -> must be 200
```

**Launching the built .exe requires `-WorkingDirectory <projectdir>` and `--urls`**, else
appsettings.json is not found (every request 500s on a missing connection string) and the
API listens on 5000. Servers must be started detached or the tool's process cleanup kills
them seconds after boot.