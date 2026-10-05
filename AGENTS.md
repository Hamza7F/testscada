# AGENTS.md â€” scada_demo Project Map

> **Read this file FIRST before making any change.** This file is a complete map of
> the project, its conventions, and its architecture. Do NOT re-scan the repo to
> orient yourself â€” read this, then only open the files you touch.

## What this project is

A production-style SCADA / IIoT Device & Sensor Orchestration Platform. Physical
gateways (Norvi ESP32, USR-W610) are registered by operators from the web UI; slave
sensors (Kaifeng Thermal Mass Flowmeter, Aosong AQ3485) are added against a **driver
registry**; a background Modbus polling worker reads the gateways over Modbus TCP
and streams each sensor's values into its **own dynamically-provisioned telemetry
table** in PostgreSQL.

| App | Tech | Location |
|---|---|---|
| API | ASP.NET Core 10, EF Core 10 + Npgsql, JWT, SignalR | `src/scada_demo_test.API` |
| Web dashboard | .NET 10 Blazor Server (Cookie auth â†’ API JWT) | `src/scada_demo_test.Web` |
| Mobile | .NET MAUI (Android/Windows) | `src/scada_demo_test.Maui` |
| Mobile (alt) | Expo / React Native | `mobile/` |
| Firmware | ESP32 Arduino (PlatformIO, ModbusMaster) | `firmware/Norvi-ESP32` |

**All simulation / mock loops have been purged** (no demo devices, no fake
generators). The dashboard shows real data only; offline hardware is shown as
OFFLINE gracefully and never crashes the worker.

## Solution layout (scada_demo_test.slnx)

6 projects, all `net10.0`:

```
src/
  scada_demo_test.Domain/          Entities, Enums, Constants, RollupMath,
                                   Drivers/ (ISensorDriver + 2 drivers), FirebaseScadaService
  scada_demo_test.Application/     DTOs, service interfaces + TelemetryIngestService,
                                   RollupCompressionService, ReportQueryService
  scada_demo_test.Infrastructure/  EF DbContext + repos, Identity (JWT), migrations,
                                   Modbus/ (ModbusTcpMaster + ModbusPollingHostedService),
                                   SignalR hub/broadcaster, AlertService, rollup hosted
                                   service, DynamicSchemaInitializer
  scada_demo_test.API/             ASP.NET controllers (entry point, binds everything)
  scada_demo_test.Web/             Blazor Server UI (talks to API over HTTP; JWT forwarded)
  scada_demo_test.Maui/            MAUI app (own services, Firebase-based, independent)
```

### Dependency direction

`API â†’ Infrastructure â†’ (Application â†” Domain)` and `API â†’ Application`.
`Web â†’ Domain` ONLY (AppTabs/AppPermissions string constants; everything else over HTTP).
`Maui â†’ Domain` (FirebaseScadaService) + its own services; does NOT use the API.

## IIoT orchestration core (MUST understand before touching)

- **Device = physical gateway/controller** (Norvi ESP32 or USR-W610). Entity `Device`
  carries comm settings: `HardwareType`, `IpAddress`, `Port`, `BaudRate`, `Parity`,
  `StopBits`, `TimeoutMs`, `MaxSensorCapacity` (hard limit **25**), `IsOnline`.
- **Sensor = slave meter/transducer** on the device's RS-485 bus. Entity `Sensor`
  stores `SensorTypeKey` (driver registry key), `SlaveAddress`, `PollIntervalSeconds`,
  `CalibrationMultiplier`, `TelemetryTableName`, `IsActive`, `IsOnline`.
- **Driver registry** (`Domain/Drivers/`): `ISensorDriver` is the isolated spec +
  parser per sensor family. Installed drivers:
  - `AosongAQ3485Driver` â€” start reg 0x0000, 2 registers, signed16/10 â†’ Â°C + %RH.
    Columns `TemperatureC`, `HumidityRH`.
  - `KaifengFlowmeterDriver` â€” start reg 0x0001, 12 registers, IEEE-754 float32
    (regs 1-2 â†’ FlowRate mÂ³/h, regs 3-4 â†’ Totalizer mÂ³). Columns
    `InstantaneousFlowRate`, `AccumulatedTotalizer`.
- **Dynamic telemetry tables**: adding a sensor provisions
  `telemetry_sensor_{type}_{sensorId8}` via `SensorTelemetryRepository`
  (`CREATE TABLE IF NOT EXISTS`), columns: `Id`, `TimestampUTC`, `RawHexBuffer`,
  `<driver.PrimaryColumnName>`, `<driver.SecondaryColumnName>`, `ConnectionStatus`
  (0 offline / 1 online), `ErrorCode`. Column/table names ALWAYS come from the
  driver catalog + `TelemetryTableNaming` (validated `[a-z][a-z0-9_]*`) â€” never
  user input.
- **ModbusPollingHostedService** (`Infrastructure/Modbus/`) polls every cycle
  (1s tick), opens ONE TCP session per gateway (`ModbusTcpMaster.OpenAsync`),
  reads holding registers (FC03) for each due sensor (respects PollIntervalSeconds),
  runs the driver parser (Ã— CalibrationMultiplier), inserts into the isolated table.
  Connect/read failures mark device + sensors OFFLINE and never throw out of the loop.
  Offline telemetry rows are only written on an onlineâ†’offline transition.

## API entry points

- **API Program.cs** â€” DI, JWT, RBAC (`Tab:*` / `Action:*`), hosted services
  (ModbusPollingHostedService, RollupCompressionHostedService), SignalR hub
  `/hubs/telemetry`, `MigrateAsync()` + `IdentitySeeder.SeedAsync()` +
  **`DynamicSchemaInitializer.EnsureSchemaAsync`** (idempotent `ADD COLUMN IF NOT
  EXISTS` for the new Device/Sensor columns â€” no dotnet-ef migration file).
- **DevicesController** (`/api/devices`) â€” gateway CRUD with hardware type,
  comm settings, capacity + sensor counts. POST requires `Action:Devices.Add`.
- **SensorsController** (`/api/sensors`) â€” `GET /libraries` (installed driver
  catalog for the Add-Sensor form), `GET ?deviceId=`, POST creates the sensor +
  provisions its telemetry table, `GET /{id}/telemetry`, DELETE unregisters
  (historical table kept).
- **TelemetryController** (`/api/telemetry/tank-reading`) â€” Norvi ESP32 firmware push
  ingest (**primary Norvi path**). Attributed by source IP â†’ `Devices.IpAddress`
  (fallback `deviceId`/`externalId`); matched to sensors by `SlaveAddress == tank_id`;
  unknown devices/sensors dropped with 200 (`unknown_device`/`unknown_sensor`).
  The USR-W610 path stays on `ModbusPollingHostedService`. See CURRENT WORK section.
- **AuthController** (`/api/auth/login`) â€” JWT identity. Web login proxies to this.

## Web dashboard

- Login (`Pages/Login.razor` â†’ `/account/login` handler) calls the **API** for the
  JWT, plants `perm` claims; `PermissionForwardingHandler` forwards Bearer token to
  API calls.
- `Pages/Devices.razor` = **Device Operations Dashboard** â€” gateway cards with
  ONLINE/OFFLINE badges, `Sensors X/25` capacity, expandable sensor tree (live
  values), `+ Add Device` modal, cascading `+ Add Sensor` modal (device â†’ driver
  library â†’ pre-filled Modbus fields with validation, capacity warning). Auto-refresh
  every 4 s.
- `ScadaDemoTestApiClient` (Web/Services) talks to the API for devices/sensors/login.
  Its Firebase-backed methods (users/roles/sites/audit) remain for legacy pages.

## Config / secrets

- API DB connection in `src/scada_demo_test.API/appsettings.json` â†’ **local SQL
  Server LocalDB** (switched 2026-09-22): `Server=(localdb)\MSSQLLocalDB;Database=scada_db_iiot;Trusted_Connection=True;TrustServerCertificate=True;`
  Schema is created by the fresh `InitialSqlServer` EF migration (`MigrateAsync`
  at startup) + `DynamicSchemaInitializer` + `IdentitySeeder`. **Supabase (Postgres)
  is fully commented out, not deleted** â€” revert path in `DB_LOCAL_SWITCH_CHANGELOG.md`,
  old Postgres migrations backed up under `backup_postgres_migrations/`.
  (Legacy Supabase reference, kept as comment only: session pooler
  `aws-0-ap-south-1.pooler.supabase.com`, project `ampzxkdfzrylnpqbzpzs`,
  user `postgres.ampzxkdfzrylnpqbzpzs`, port 5432, `SSL Mode=Require`.
  The direct host `db.ampzxkdfzrylnpqbzpzs.supabase.co` is IPv6-only.)
  Do not commit new credentials.
- **Provider sensitivities after the SQL Server switch**: raw SQL that touches
  custom tables lives ONLY in `EfResilience` (SqlException),
  `SensorTelemetryRepository` and `DynamicSchemaInitializer` (T-SQL dialect â€”
  `datetime2`/`float`/`nvarchar(max)`/`uniqueidentifier`/`bit`,
  `OBJECT_ID(...) IS NULL` `COL_LENGTH(...) IS NULL` guards, `SYSUTCDATETIME()`,
  `TOP`, `IDENTITY(1,1)`). Everything else is EF LINQ (provider-agnostic).
  New EF migrations MUST be generated for the SqlServer provider
  (`dotnet ef migrations add X --project src/scada_demo_test.Infrastructure
  --startup-project src/scada_demo_test.API`).
- Web `appsettings.json`: `ApiBaseUrl` = `http://localhost:5080` (the API, not the
  Web app). Signals the dashboard's data source.
- Firebase config (DB URL / API key / project id) hardcoded in
  `Domain/Services/FirebaseScadaService.cs` (`scada-monitoring-system`).
- JWT defaults: key `SCADA_ENTERPRISE_SUPER_SECURE_SECRET_KEY_2026_...`,
  issuer `AlamIotScadaApi`, audience `AlamIotScadaClients`.

## Domain facts / conventions

- **Device IDs**: `DEV-XXXXXXXX` (auto). Legacy `norvi-slave-{id}` still accepted for
  existing data.
- **Sensor IDs**: `S-XXXX` (auto, unique).
- **Status**: devices/sensors expose `IsOnline` (worker-maintained). Legacy `Status`
  enum still present but secondary.
- **RBAC**: `AppTabs.All` and `AppPermissions.All`; policies `Tab:{name}` /
  `Action:{name}`; SuperAdmin bypasses via `IsSuperAdmin=true` or role `SuperAdmin`.
- **DB hot-path writes** wrapped in `EfResilience` (Polly retry) â€” including
  telemetry table inserts.
- **Rollup pipeline** (legacy SensorReadings) still runs: Raw â‰¤3h â†’ Hourly (â‰¤7d) â†’
  Daily (â‰¤60d) â†’ Monthly. Only `Totalizer` is cumulative
  (`MetricCatalog.IsCumulative`).
- API listens on `0.0.0.0:5080`; Web on `localhost:5150`.

## Build / run

```
# API (port 5080, binds 0.0.0.0)
dotnet run --project src/scada_demo_test.API

# Web (port 5150)
dotnet run --project src/scada_demo_test.Web

# Mobile (Expo/RN)
cd mobile && npm start

# Firmware (PlatformIO)
cd firmware/Norvi-ESP32 && pio run -t upload
```

- No unit test projects.
- Schema changes to `Devices`/`Sensors` are additive via `DynamicSchemaInitializer`
  (extend that list, NOT the EF migration flow). It is safe on every boot: it also
  provisions the whole `Sensors` table (`CREATE TABLE IF NOT EXISTS` + indexes + FK),
  because legacy cloud DBs predate the `InitialPostgres` migration.
- `MigrateAsync` at startup is best-effort: if it collides with a pre-existing schema
  (`PendingModelChangesWarning` / already-existing tables) the API logs a warning and
  continues with the additive schema upgrade + seeder â€” never refuses to boot.

## Gotchas

- `src/scada_demo_test.Maui.rar` is a backup archive â€” do not treat it as source.
- `Must ignore in diffs / never edit`: `obj/`, `bin/`, `.vs/`, `mobile/node_modules/`.
- Firmware `lib/network/network.cpp` contains a real WiFi password â€” never leak into
  logs/commits.
- Solution uses `.slnx` (new XML solution format). Building the whole slnx is slow
  because of the MAUI project â€” build API/Web projects directly for fast feedback.
- **CRITICAL â€” static web assets / dead-UI gotcha (FIXED IN CODE 2026-09-26).**
  Launching `scada_demo_test.Web.exe` directly (no `ASPNETCORE_ENVIRONMENT`) defaults to
  `Hosting environment: Production`, and in Production the static web assets manifest is not
  loaded automatically, so `/_framework/blazor.server.js` returned **404**. Symptom is
  deceptive: every page prerendered and looked perfect (HTTP 200) but the Blazor circuit
  never started, so **no button, dropdown, tab or logout worked** â€” the UI looked frozen.
  `Program.cs` now calls `builder.WebHost.UseStaticWebAssets()` (guarded by a manifest
  file check) so this is now **environment-independent**. Verified by starting the exe
  with no environment variable: `Hosting environment: Production` yet
  `/_framework/blazor.server.js` â†’ **200**, `POST /_blazor/negotiate` â†’
  `availableTransports: WebSockets, ServerSentEvents, LongPolling`, and a real WebSocket
  connect â†’ `Open`. Still pass `ASPNETCORE_ENVIRONMENT=Development` when launching so hot
  reload etc. behave normally, but a wrong launch can no longer break the UI.
  Quick health check any time:
  `curl -o NUL -w "%{http_code}" http://localhost:5150/_framework/blazor.server.js` â†’ must be **200**.
- MAUI project has helper `MauiDiag.cs`; `Platforms/Android` obj caches may say
  `net9.0-android` but the csproj targets `net10.0`.
- Deleting a Sensor keeps its telemetry table (historical data preserved).
  Deleting a Device cascades its Sensor rows.

---

# CURRENT WORK â€” HANDOVER FOR NEXT AI

> ## !! 2026-10-01 - SOURCE LOSS + RESTORE IN PROGRESS - READ FIRST !!
> The entire `src/` tree was wiped (cause unknown; Recycle Bin empty, no shadow
> copies available). The repo root survives (this file, DB_LOCAL_SWITCH_CHANGELOG.md,
> .slnx, backups). `src/` was restored from `C:\dx main\scada_demo (5).zip`
> (2026-09-22 16:07), so the **code is at the 2026-09-22 state** while the LocalDB
> `scada_db_iiot` schema/data is at the **2026-09-30 state**. Every step lazily built
> between 2026-09-22 evening and 2026-09-30 is MISSING from source and MUST be
> re-implemented: items 25, 26-28, 29-32, 33-37, 38, 39, 40-45. This AGENTS.md is the
> spec for that work. Checkpoints: source-only backup
> `C:\dx main\scada_demo_src_sourceonly_10-01.zip` (98 MB, 223 files); a git repo was
> initialized in the root (initial commit d31861f). A stray unrelated `dy` file (CoreUI
> React HTML, 2026-09-30 16:19) sits at the root - NOT part of this project.
>
> RE-IMPLEMENTED + VERIFIED SO FAR:
> - **item 25 (DB switch Supabase -> local SQL Server LocalDB) - DONE 2026-10-01**:
>   Infrastructure csproj `Npgsql.EntityFrameworkCore.PostgreSQL` ->
>   `Microsoft.EntityFrameworkCore.SqlServer` 10.0.11; appsettings
>   `Server=(localdb)\MSSQLLocalDB;Database=scada_db_iiot;Trusted_Connection=True;TrustServerCertificate=True;`
>   (Supabase string kept as a comment); Program.cs `UseNpgsql` -> `UseSqlServer` and the
>   old Supabase-era "stamp InitialPostgres history" baseline DELETED (MigrateAsync alone);
>   EfResilience `NpgsqlException` -> `SqlException` + retry 4x exponential-200ms ->
>   3x fixed-100ms; SensorTelemetryRepository + DynamicSchemaInitializer fully rewritten to
>   T-SQL (`bigint IDENTITY`, `datetime2`/`SYSUTCDATETIME()`, `float`, `nvarchar(max)`,
>   `smallint`, `bit`, `uniqueidentifier`, `OBJECT_ID`/`COL_LENGTH`/`sys.indexes`/
>   `sys.foreign_keys` guards, `TOP (@n)`); Postgres migrations moved out (already in
>   `backup_postgres_migrations/`) and `20260922112111_InitialSqlServer` re-generated for
>   SqlServer (renamed to the exact id already in `__EFMigrationsHistory` so MigrateAsync
>   reports "already up to date"). Also recreated the missing `SensorReadings` table
>   (present in the migration, absent in the 9/30 DB) so the legacy rollup worker no-ops
>   silently instead of logging `Invalid object name 'SensorReadings'` every tick.
>   VERIFIED: API boots, `No migrations were applied. The database is already up to date.`,
>   libraries = 3 drivers, login returns a 1183-char token, devices = 3, zero fail/error
>   in the API log.
>
> Last updated: **2026-10-01 (re-implementation in progress)**. Items 29-32 + the
> scanner rewrite (33-45) are now in source and **VERIFIED LIVE on the real USR-W610
> bus** (see below). The stale 2026-09-30 excerpt immediately below is the ORIGINAL
> spec text; the live bus map in those lines is now superseded by the measured map
> under "LIVE SCAN VERIFIED 2026-10-01".
>
> - **Items 29-32 (driver contract + Vortex/Selec drivers) - DONE 2026-10-01**:
>   `ISensorDriver.cs` extended with `SensorReadWindow` / `WindowTelemetry` /
>   `ParsedTelemetry` + default `ReadWindows` / `ParseWindow` / `CorroborationWindow`;
>   `ModbusValueCodec.cs` NEW (high/low-word-first float + negative-zero fold);
>   `AosongAQ3485Driver.cs` reg order fixed (reg0=humidity, reg1=temp);
>   `KaifengFlowmeterDriver`/`ElectromagneticFlowmeterDriver` refactored to the codec;
>   `VortexFlowmeterDriver` (VORTEX_FLOWMETER, FC04 @1026 q2, proof @1067) and
>   `SelecPowerMeterDriver` (SELEC_POWER_METER, FC04 @42 q2, proof @64 q10, low-word-first)
>   NEW, both registered in `SensorDriverCatalog` + `Program.cs` DI. `SmartScanDtos.cs`
>   rewritten (`Passes`/`DeepDuplicateCheck`, `ScannedCandidateDto`, `IsAmbiguous`/
>   `Candidates`, `DuplicateSlaveIdConflictDto`, `AmbiguousSlaves`/`DuplicateIdConflicts`).
>   `ModbusTcpMaster.cs` gained `ModbusException.IsProtocolError` + a new
>   `ModbusConnectException` (connect failure is a gateway-wide fault, not a silent
>   slave). `ModbusPollingHostedService.cs` now loops `driver.ReadWindows`, dispatches
>   FC03 vs FC04, merges nullable primary/secondary, concatenates raw, applies
>   calibration once, and applies `InterWindowDelayMs=60` between windows.
> - **`ModbusScanner.cs` FULL REWRITE (items 31-45) - DONE 2026-10-01**: presence probe
>   (FC03 @0 + FC04 @42), every driver's own identification window, remaining windows,
>   corroboration with `DegeneracyProofAttempts=4`, degeneracy guard, rank
>   `(Width, Unproven, Live, Extra, Order)`, vote gate + foreign-window collision hunt,
>   tiering (Quick 1..20, Full 1..20 then 21..247), rank-aware merge, ambiguity flag,
>   per-address isolation + `UnreachableFailureStreak=12`, scan connect capped at 1500ms.
>   `DevicesController.ScanBus` extended (`Passes`, `DeepDuplicateCheck`, EndAddress=247)
>   and a new `GET /api/devices/{id:guid}/reachability` endpoint added.
> - **LIVE SCAN VERIFIED 2026-10-01** on `USR Gateway` (DEV-A355817F, USR-W610,
>   10.10.100.254:502): **Quick (`passes=1, deep=false`) = 18s, found=4/4, conflicts=0,
>   ambiguous=0, unknown=0**:
>   | Slave | Driver | Live |
>   |---|---|---|
>   | 1 | KAIFENG_EM_FLOWMETER | 0 m³/h / 0.64 m³ |
>   | 2 | VORTEX_FLOWMETER | 0 m³/h / 0 m³ |
>   | 3 | AOSONG_AQ3485 | 29.1 °C / 56.8 %RH |
>   | 4 | SELEC_POWER_METER | 0 kW / 0 kWh |
>   **Full (`passes=0, deep=true`) = 243s, found=4/4, conflicts=0** - but slave 4 is
>   flagged `ambiguous` in that one run because the Selec's `@64 q10` proof read failed
>   once, tying the Kaifeng ghost on Width+Unproven. Quick is clean; Full has that one
>   false ambiguity (the gateway was OFF for part of the session, so the first Quick
>   run correctly returned `responding=0` in 200s until the connect cap was added).
>   `GET /api/devices/{guid}/reachability` verified: offline gateway -> 1506ms
>   `reachable=false`; Norvi -> data-driven message, no TCP probe. API :5080 (PID 2936)
>   + Web :5150 both running, `/_framework/blazor.server.js` -> 200, login -> 302
>   `/dashboard`, all routes 200 with the cookie jar. Builds 0 warnings/0 errors.
>   - **DUPLICATE DETECTION VERIFIED 2026-10-01** with purpose-built mocks:
>     - **dupmock** (3 families sharing slave 2): Quick `conflicts=1, meters=3, found=0`;
>       Full same. Message: "Your 3 sensors are using the same slave ID 2. Change all but
>       one to a different slave ID, then scan again." Correct — colliding address hidden
>       from Found, conflict count accurate.
>     - **uniqmock** (unique zero-filler Selec at slave 5): `found=1 (SELEC_POWER_METER),
>       conflicts=0, ambiguous=0`. No false positive on the S5 scenario the framing path
>       used to trip on. Both vote-gate AND foreign-window paths verified.
>   - **STILL OPEN**: Web UI still maps the OLD scan DTO shape (no
>     `IsAmbiguous`/`Candidates`/`DuplicateIdConflicts`/`Reachability` in
>     `Gateway.razor`); `AGENTS.md` item-45 bus map says S1=Aosong/S2=Vortex/S3=EM/S4=Selec
>     which is now superseded by the measured map above (S1=EM/S2=Vortex/S3=Aosong/S4=Selec).
>
> The 2026-09-30 spec excerpt below is kept for reference; the live bus map in it
> (1=Aosong, 2=Vortex, 3=EM, 4=Selec) is superseded by the measured map above.
> now an opt-in cost instead of a mandatory one; the on-screen estimate is an upper
> bound that stops quoting a number once real time passes it.
> PREVIOUS: 2026-09-30 (quick-scan SPEED work + bus re-map - see item 44). **The bus
> layout CHANGED while we were testing: the operator moved the EM flowmeter off the
> contested slave 2 onto its own slave 6, so there is NO duplicate any more - the
> current map is 1=Aosong, 2=Vortex, 3=EM, 4=Selec and the scan correctly reports
> `found=4, conflicts=0`.** The duplicate detector itself is unchanged and still
> proven by the mocks. Quick scan went 188 s -> ~165 s with ZERO change to what is
> found (identical sample counts and identical decision rule); the stale "~15 s"
> estimate that was off by 10x is now derived from measured constants. The bus
> re-layout means slave numbers differ between older log excerpts in this file - trust
> the live scan, not an old number.
> PREVIOUS: 2026-09-30 (duplicate slave-ID detection COMPLETE - see item 43). **The
> physical bus is slave 2 = EM + vortex (TWO meters on one ID), slave 3 = Aosong,
> slave 5 = Selec** - item 42's "slave 2 holds only the EM" was wrong and the operator
> confirmed it. The scan now returns `found=2` (S3 Aosong + S5 Selec) + one hidden
> conflict for slave 2 with `meterCount=2`, **6/6 live runs identical** (5 narrow
> 108-122 s + one full `1..247` at 155 s), using a NEW second trigger beside the vote
> gate: the **foreign-window reply RATE** (0% = nobody serves it, ~100% = the address's
> own meter or a zero-filler owns it, a FEW PERCENT = one device is racing another).
> Narrow q2 windows only, winner's own window skipped, 300-sample budget, 15-read
> connection reuse, plus a give-up rule so a clean address costs ~40 samples per window.
> The `framingConflict` byte-count path stays log-ONLY and must NEVER become a trigger
> again (it hid the unique S5 in 4/8 runs). Also fixed: slave 5 was misidentified as
> `AOSONG_AQ3485` in ~1 of 3 runs because the Selec genuinely reads 0 kW/0 kWh (CT not
> connected), so the degeneracy guard rejected it before the `Unproven` tie-break ran
> and the AQ3488 ghost (its CT primary rating 1000, 0 as `HumidityRH=100,
> TemperatureC=0`) won the pass - the guard now RE-READS the proof block up to 4 more
> times before rejecting (`DegeneracyProofAttempts`; the read measured 60/60 live on the
> real bus, so the old 2 attempts were just losing genuine matches to transport
> hiccups). All three regression mocks pass: unique zero-fill `found=1 conflicts=0`
> (no false positive), shared-3 `meters=3` via the vote gate, same-driver `meters=2` via
> the new foreign trigger - i.e. same-driver duplicates are now caught too, which is
> BETTER than item 41 believed possible. API :5080 + Web :5150 are running with NO
> Debug logging; re-enable with `set Logging__LogLevel__Default=Debug` when per-address
> scoring evidence is needed).
> PREVIOUS: 2026-09-29 (duplicate slave-ID vote gate FINAL: the DECISION was the STRONG
> vote gate ONLY (`collidingFamilies.Count >= 2`) - the `framingConflict` byte-count path
> was removed as a standalone trigger because the USR-W610 bridge intermittently garbled a
> UNIQUE meter's frames and hid the unique S5. It was re-tested at **6/6 unanimity** and
> STILL hid the healthy S5 in 4 of 8 runs, so it is computed for the log line ONLY and can
> never be re-added as a trigger. Unique slave is now ALWAYS found (real bus 8/8 runs
> `conflicts=[]`); a genuine distinct-family duplicate is still hidden with an accurate
> count, and **add-time slave uniqueness is enforced server-side for ANY driver** (HTTP
> 400). Conflict message simplified to one line: "Your {n} sensors are using the same slave
> ID {slave}. Change all but one...". Verified live: real bus S2+S5 found with 0 conflicts;
> shared-3 mock count=3; unique zero-fill mock found; **same-driver mock count=2** (this is
> the framing signal working in isolation, and is also why it cannot be trusted on the real
> bus) - see item 41).
> PREVIOUS: bus scan rewritten: **AutoPasses 3â†’1** because the
> 741-probe sweep was wedging the USR-W610 bridge, plus **targeted re-probe of only
> the responding addresses** so identification stays accurate; **duplicate slave-ID
> detection** that hides a colliding address and tells the operator to change an ID;
> **4/4 meters now found with the correct driver, 3/3 identical scans** â€” see item 40).
> LocalDB `scada_db_iiot` + **reading-delay tuning**: Supabase-era 4x exponential
> EfResilience retry â†’ 3x fixed-100ms, see DONE item 25 + `DB_LOCAL_SWITCH_CHANGELOG.md`;
> login logo 140px + sidebar logo.jpg branding + collapsed nav groups; previously
> TREND parent group + Plant Energy arc-gauge dashboard).
> PREVIOUS: UX cleanup round â€” dead tabs removed, fast reliable modal toasts, in-app
> alert notification toast, Reports PDF export restored, Charts overhaul with time
> axis/level chips/KPI strip, formal login).
> PREVIOUS: alert rule engine wired into the poller; slow-bridge reconnect gated).
> PREVIOUS: Smart & Fast RS-485 bus scan (on-the-fly meter type detection, 20-200ms
> probes, auto-skip registered slaves, one-click map into pre-filled Add Sensor form;
> third driver Kaifeng IEMFL Electromagnetic water flowmeter from ModbusMeterLibs).
> DEVICE TAB SPLIT (2026-09-21): Devices is now a parent nav group â€” children
> "Norvi" (`/devices`, restored to its pre-scan state) and "Gateway" (`/gateway`,
> the full Add Gateway â†’ Find Sensors â†’ multi-select â†’ Configure â†’ Add workflow).
> TREND GROUP (2026-09-22): The arc-gauge plant overview is a **TREND CHILD**
> (`/dashboard` a.k.a `/energy`, nav label "Plant Energy"). Charts & Storage Tanks
> are TREND children too. Live Monitoring (`/`) and Totalizer Summary (`/summary`)
> are top-level. See step 23 for the Dashboard.
> If another AI takes over, this is the single source of truth for "where we are"
> and "what to do next". Read this entire section before touching any code. Keep
> it updated as you make progress.

## CURRENT INVESTIGATION - SCAN TIMEOUT MESSAGE (2026-09-29)

User reported the Gateway page showing "The scan did not finish in time (about 15 s needed)" on every scan. Extensive live verification eliminated every server-side cause:

- Server scan (exact Web payload `{probeTimeoutMs:200,startAddress:1,endAddress:247,passes:1}`) returns HTTP 200 in 12-21 s with a full, correct payload - even with a live duplicate-ID conflict on the bus (duplicateIdConflicts populated, S5 = Aosong conflict) - verified 2026-09-29 13:0x.
- Web DTOs (GatewayScanResultDto / ScannedMeterDto / DuplicateSlaveIdConflictDto in ScadaDemoTestApiClient 602-632) EXACTLY match Application/DTOs/SmartScanDtos.cs server records field-for-field incl. DetectedDriverKey / DetectedDriverName / Message.
- Web DI binds _scanClient = 5-min client (Program.cs BuildApiClient, AddScoped 138-141); ScanGatewayBusAsync (898-930) sizes budget = (count*(probeTimeout+150)*sweepCount)+30000 = 116 s for quick, far below server 21 s, so client timeout cannot fire on a healthy API.
- Running binaries confirmed CURRENT at 13:02/13:03 (Web dll 13:02:57, new PID 34160 started 13:31:11; API dll 13:02:13, PID 14392). Logs (web-run.log / api-run.log) showed no scan exceptions - BUT those logs are from the OLD processes (stopped writing 12:55 / 13:02); current processes have no captured stdout.

Conclusion: the only remaining null path is a non-2xx or async exception inside ScanGatewayBusAsync whose `catch { return null; }` swallowed it. Added diagnostics 2026-09-29 13:30:
- ScanGatewayBusAsync catch now sets Api.ScanErrorDetail (TaskCanceledException / HttpRequestException / JsonException / raw message).
- Gateway.razor ScanErrorDetail field, reset in FindSensorsAsync, rendered under the timeout message as "IIT detail: <real reason>".
- Web rebuilt 0 warnings / 0 errors, restarted (PID 34160). Next user click on a still-failing scan shows the REAL cause instead of a guess.

If user now reports TaskCanceledException even though the server answers in ~20 s, suspect something is eating wall-clock client-side (e.g. circuit / component life).

## Current objective (ONE focus)

Build the **Norvi ESP32 firmware-push ingestion pipeline**: receive JSON pushed by
Norvi to the API, separate the two RS-485 meters (Kaifeng flowmeter slave 1,
Aosong AQ3485 slave 2), match them to registered sensors, persist telemetry, and
drive device ONLINE/OFFLINE from real data arrival. **DONE â€” implemented and
verified live (see below).**

**Why it was needed**: Norvi firmware is a Modbus **RTU master** (polls its own
RS-485 slaves), NOT a Modbus TCP server. `ModbusPollingHostedService` TCP-connects
to `IP:502`, which always fails for Norvi â†’ it showed OFFLINE. Fix = make Norvi
status **data-driven** from pushed HTTP JSON. The USR-W610 (ModbusTCP server) keeps
the poll path.

## What is DONE (verified â€” do not redo)

1. **Modal UX fixes** in `src/scada_demo_test.Web/Pages/Devices.razor` +
   `src/scada_demo_test.Web/wwwroot/css/site.css`: proper modal structure
   (header/body/footer, clip/cut fixed), inline `.modal-alert` success/error,
   success auto-closes modal ~2s, stale toast cleared on open/submit, busy guards,
   new-sensor row flash (`sensor-row-highlight`). Web + API build = 0 warnings /
   0 errors. Verified live: Web root 200, `/devices` 302, API `/api/devices` 200.
2. **Duplicate slave-address guard**: client-side block at Save click
   (`UpdateSlaveCheck()` first line of `SubmitSensorAsync`, inline message, no API
   call) + server 400 in `SensorsController.Create` â€” both verified live.
3. **Duplicate IP guard**: client-side live warning + disabled Save
   (`OnDeviceIpChanged`), server 409 in `DevicesController.Create`/`Update`
   (excludes self) â€” verified live.
4. **Push ingest pipeline (IMPLEMENTED 2026-09-16)**:
   - `Application/Services/PushTelemetryIngestService.cs` + `IPushTelemetryIngestService`
     + `PushTelemetryMessageDto`/`PushTelemetryReadingDto` (Application/DTOs):
     attributes device by **source IP** â†’ `Devices.IpAddress` (fallback payload
     `deviceId`/`externalId`), matches sensor by `SlaveAddress == tank_id`, maps
     metric alias â†’ driver column (see alias map), inserts telemetry row with
     `ConnectionStatus=1`, sets `LastSeenAt` + `IsOnline=true` on device + sensor.
   - `TelemetryController.TankReading` rewritten: no more `norvi-slave-{tankId}`
     external-id scheme; returns 200 `{status:"unknown_device"}` /
     `{status:"unknown_sensor"}` / `{status:"accepted",...}`. `NormalizeIp()`
     strips the `::ffff:` IPv4-mapped prefix. Anonymous endpoint (firmware has no
     auth). **Auto-registration does NOT happen**: `TelemetryIngestService` drops
     unknown external ids (verified) â€” devices must exist first (UI).
   - `IDeviceRepository.GetByIpAddressAsync` added (Infrastructure impl exists).
   - Verified live end-to-end (11 real devices in DB + a temp loopback test device):
     unknown-source push â†’ `unknown_device`; push `tank_id=1` (Flowrate/Totalizer)
     and `tank_id=2` (Temperature/Humidity) â†’ `accepted`, device + both sensors
     ONLINE, telemetry rows with correct values in the per-sensor tables; unknown
     metric (`Pressure`) skipped (`skippedReadings:1`) while known one persisted.
5. **Online watchdog (IMPLEMENTED 2026-09-16)** in
    `Infrastructure/Modbus/ModbusPollingHostedService.cs`:
    - Every 1s cycle: offline if `device.IsOnline && (now - LastSeenAt) > timeout`
      (`timeout = 2x` slowest active sensor poll interval, clamped 5s..20s).
      Exactly ONE offline row written per real onlineâ†’offline transition
      (`RecordWatchdogOfflineAsync`, errorCode `NO_DATA_TIMEOUT`).
    - **NorviESP32 devices are skipped by the TCP poller entirely**
      (`HardwareType == NorviESP32` â†’ `continue`): no more futile TCP connects;
      their online state is 100% LastSeenAt-driven.
    - TCP connect/read failure never flips a device offline while `LastSeenAt` is
      fresh (guards in backoff + finally branches).
    - Verified live: after ~10-20s without pushes the test device flipped
      OFFLINE with exactly one `NO_DATA_TIMEOUT` row per sensor, then back ONLINE
      on the next push.
6. **UI**: Device status sub-bar added to each device card in `Devices.razor`
    (shows DEVICE ONLINE/OFFLINE badge, Sensors X/Y count, Last seen time).
    Added between card header and expanded section.
7. **Device dropdown removed** from Add Sensor modal in `Devices.razor`
    (device implied by the expanded device card; static `Target Device` text +
    `SelectedDevice` computed property; `OnSensorDeviceChangedAsync` removed).
    Sensors can only be added from an expanded device row now.
8. **Smart & Fast RS-485 bus scan (IMPLEMENTED 2026-09-21)** â€” device tab workflow:
   Step 1 Gateway Add â†’ Step 2 scan (1-by-1, 20 ms probes) â†’ Step 3 on-the-fly type
   detection (no DB write) â†’ Step 4 pre-filled one-click "Map as Sensor":
   - **New scan endpoint** `POST /api/devices/{id}/scan` (`ScanBus` in
     `DevicesController`, `[Authorize Action:DevicesView]`). Requires the gateway IP
     (400 if unset); skip list = slave addresses already attached to the device;
     returns 502 `GATEWAY_UNREACHABLE` on connectivity failure (with the partial
     `scan` payload inside the response body).
   - **`ModbusScanner`** (`Infrastructure/Modbus/ModbusScanner.cs`, `ISmartScanService`
     in `Application/Interfaces`, DTOs in `Application/DTOs/SmartScanDtos.cs`). Opens
     ONE `ModbusTcpSession`, pings slaves 1..247 sequentially, per-probe read timeout
     clamped 10..1000 (default 20 ms), auto-skips registered slaves. Meter type is
     identified **on the fly** from the register signature: each installed driver's
     window is read in order (AQ3485 start 0 qty 2 â†’ plauible 0..100 %RH & -60..150 Â°C;
     Kaifeng start 1 qty 12 â†’ finite sane floats; EM start 90 qty 10) and `ParseData`
     must satisfy a plausibility check. Exception frame 0x0B (gateway target failed
     to respond) â†’ slave absent; other exceptions (e.g. 0x02) â†’ try next signature;
     timeout/IO â†’ absent; data-but-no-match â†’ `unknown` responder (surfaced, not
     mappable). Returns `found[]` with live values + `skippedSlaves` + counts.
   - **Per-read timeout**: `ModbusTcpSession.ReadHoldingRegistersAsync` gained an
     optional `readTimeoutMs` (polling worker unaffected, keeps session default).
     `ModbusException` now carries `ExceptionCode` (nullable).
   - **New driver #3**: `ElectromagneticFlowmeterDriver` (`KAIFENG_EM_FLOWMETER`,
     simple `kaifeng_em`, start reg 90 qty 10, float32 totalizer@90-91 + flow@98-99,
     mÂ³/h + mÂ³) extracted from `C:\Users\pc\Downloads\ModbusMeterLibs` â€”
     registered in `SensorDriverCatalog` + DI (`/api/sensors/libraries` now lists 3).
- **Web**: `ScanGatewayBusAsync(deviceId, probeTimeoutMs=20)` in
      `ScadaDemoTestApiClient`; the scan UI (button + results panel) now lives in
      `Pages/Gateway.razor` â€” see item 10 (the Devices-page version from 2026-09-21
      earlier in the day was removed). Only identified meters get mapped (pre-fills
      `SensorTypeKey`/`SlaveAddress`/`SuggestedName`/poll interval, duplicate +
      capacity checks intact). Verified LIVE against a local mock Modbus TCP gateway
     (127.0.0.1:12502, C# mock in `%TEMP%\opencode\modbusmock`): pre-registration
     scan reported slave 1 = Kaifeng (live 2.5/123.4) + slave 2 = AQ3485 (25/42)
     with no DB rows; after registering both slaves the rescan auto-skipped them
     (`skippedSlaves 1,2`, found=0). Cleanup verified (device + tables dropped).
9. Both projects build 0 warnings / 0 errors (API + Web, `.slnx` NOT built â€”
    MAUI is slow). **API (:5080) and Web (:5150) are running right now** (session
    background processes) so the next AI can curl immediately.
9b. **0-sensor gateway ONLINE probe (IMPLEMENTED 2026-09-21)**: 
    `ModbusPollingHostedService` used to `continue` when a USR-W610 had no active
    sensors â†’ it could NEVER go ONLINE even when reachable. Fixed: when `dueSensors`
    is empty the worker now probe-connects TCP every cycle (no backoff), sets
    `IsOnline=true` + refreshes `LastSeenAt` (throttled to every >10s) on success and
    marks OFFLINE on failure. So a freshly-added gateway shows ONLINE from pure
    reachability before any sensor is attached. Verified live: US_TEST (USR-W610 @
    10.10.100.254:502, 0 sensors) flipped ONLINE. NOTE: machine has dual NICs â€”
    Ethernet 2 = internet (192.168.100.128), Wi-Fi = USR-W610 LAN (10.10.100.100,
    gateway 10.10.100.254). Route preference: default via internet NIC; USR on its
    own subnet is direct â€” no conflict, DB stays reachable.
11. **REAL-BUS live validation + FAST scan (2026-09-21)** (USR-W610 @ 10.10.100.254):
    - Temp sensor round-trip is **~90-105 ms** (9600 baud over serialâ†’WiFi) â€” the old
      20 ms probe ALWAYS missed it. New default probe timeout = **120 ms** (UI input,
      server clamps 10..1000).
    - Async reads ignore `TcpClient.ReceiveTimeout` â†’ probe reads could hang forever
      and empty slaves threw spurious `SocketException` â†’ GATEWAY_UNREACHABLE.
      `ModbusTcpSession.ReadExactlyAsync` now enforces a REAL hard deadline via linked
      `CancellationTokenSource.CancelAfter(timeoutMs)`; on timeout it `DrainPendingBytes()`
      so the shared session stays clean (fast scans continue WITHOUT re-opening).
    - Scanner re-opens the TCP session only on genuine socket failure, never on a
      plain probe timeout (that reconnect-per-silent-slave is what made scans crawl).
    - **Address range scan**: `SmartScanRequest` has `StartAddress`/`EndAddress`
      (clamp 1..247). **Default is now the FULL bus 1..247** (user request
      2026-09-25) in all four places: `SmartScanDtos.SmartScanRequest`,
      `DevicesController.ScanBus` (`req?.EndAddress ?? 247`), Web client
      `ScanGatewayBusAsync`, and the `Gateway.razor` field. Rationale: a silent
      slave costs only ONE read (the first signature window already returns
      Absent â†’ early return, `ModbusScanner.ProbeSlaveAsync`) + a fresh TCP
      connect, so a blank bus is ~235 ms/address
      â‡’ **247 addresses â‰ˆ 58 s** (was ~45 s at the old 20 ms probe, which
      missed 9600-baud meters). **MEASURED LIVE 2026-09-25**: full 1..247 scan
      of the real USR-W610 (10.10.100.254, 0 registered sensors) = **57.9 s**,
      `slavesScanned=247`, slave 2 identified as AOSONG_AQ3485 (45.8 Â°C / 30.3
      %RH); the slave-1 EM flowmeter was powered off (silent) at that moment.
      `Gateway.razor` now shows a live `ScanEstimateSeconds` estimate next to
      the range inputs (measured formula: addresses Ã— (probeTimeout + 35 ms) â€”
      connect overhead is only ~35 ms, NOT the 110 ms first assumed). API
      `ScanBusRequest` mirrors it. Range stays operator-editable for fast
      low-range re-scans; registered slaves are still free (auto-skipped).
    - **FC04 (input registers) support**: `ModbusTcpSession.ReadInputRegistersAsync`
      added; scanner probes each driver window with FC03 then FC04 (some flowmeters
      serve data only on input registers).
    - **Zero-ambiguity detection**: a decode of 0/0 (meters answer status registers at
      0x0000 while real data lives elsewhere) is treated as ambiguous â€” the scanner
      keeps trying later signatures and prefers a NON-zero live decode; falls back to
      the zero-match only if nothing better exists. Verified: EM meter at slave 1
      (status 0/0) now correctly identified as Kaifeng IEMFL instead of AOSONG.
    - **Multi-select no longer collapses the card**: checkbox/ClearScan/Add-button/
      probe-input clicks got `@onclick:stopPropagation` so they no longer bubble into
      the card's `SelectGateway` toggle.
    - **Verified live end-to-end**: `POST /scan` over 1..10 â†’ **2.1 s**, found
      `slave=1 â†’ kaifeng_em` (flow 0 mÂ³/h, totalizer 0.55 mÂ³) + `slave=2 â†’ aosong`
      (47.1 Â°C / 30.8 %RH), `connectivityOk=true`. (Earlier: EM meter was powered OFF â€”
      completely silent on the bus from the gateway; turning it ON made it respond.)
12. **Scan reliability + UI polish (2026-09-21)**:
    - **Per-read fresh TCP connection**: serial bridges (USR-W610) answer reliably on a
      new connection but desync under rapid back-to-back reads on ONE session (randomly
      "missing" meters / wrong fallback). Scanner now opens a fresh session per
      (slave, signature, function) read; a silent/busy address costs a single read.
      Default probe timeout bumped **120 â†’ 200 ms** (temp sensor round-trip 90-107 ms;
      120 ms intermittently missed it over WiFi). Verified: 3 consecutive scans all
      found `s1=kaifeng_em (0/0.55)` + `s2=aosong (46.9/30.7)` in ~2.9-3.2 s.
    - **Gateway list stable ordering**: `DeviceDto` gained `CreatedAt`; both
      `Gateway.razor` and `Devices.razor` now load `OrderBy(d => d.CreatedAt ?? Max).ThenBy(d => d.Id)`
      so first-added gateways stay on top across the 4 s auto-refresh.
    - **DC/ON flapping guard**: the 0-sensor reachability probe now only marks a
      gateway OFFLINE after **3 consecutive** probe failures (`_zeroSensorProbeFailures`,
      `ZeroSensorOfflineThreshold`) â€” a single WiFi blip no longer flips it OFFLINE
      for a second then back ONLINE.
13. **Offline-flapping overhaul (2026-09-21)** â€” US_TEST (1Ã— AOSONG slave 2) kept
    flipping OFFLINE/ONLINE while the user never powered the gateway off. Root causes
    + fixes in `ModbusPollingHostedService`:
    - **Per-second probe churn removed**: a gateway WITH active sensors but no sensor
      due this tick used to TCP probe-connect EVERY cycle; 3 consecutive transient
      probe failures flipped it OFFLINE even while the real 5s polls succeeded. Now
      `dueSensors.Count==0 && activeSensors.Count>0` â†’ `continue` (the next due poll
      refreshes the watchdog itself). The 0-sensor probe now runs ONLY for gateways
      with zero active sensors.
    - **Read budget too tight**: device `TimeoutMs=2000` â€” the USR-W610's serial
      bridge intermittently stalls for seconds (measured 13-27 s under contention),
      so reads timed out â†’ spurious failures. Poller now uses `ReadBudgetMs(device)`
      = `max(5000, TimeoutMs)` for connect + read + a **one-shot reconnect retry** on
      a fresh session before counting a read failure.
    - **Watchdog generous**: `ComputeWatchdogTimeout` = `max(4Ã— slowest poll, 60s)`.
      A working gateway refreshes `LastSeenAt` constantly, so a transient drop/stall
      <60s can never flip it offline; only a genuinely dead gateway goes offline.
    - **`MarkDeviceAsync` ALWAYS refreshes `LastSeenAt`** on online (was gated >15s,
      while the old watchdog was 10s â†’ the refresh could never happen in time: the
      flapping bug).
    - **Sensor 3-strike tolerance** (`SensorOfflineStreakThreshold=3`): one-off read
      blips never DC a sensor; sustained failures flip it with one offline row.
    - **Never crash, always delay**: the worker loop already catches-all per cycle, and
      now EACH device is wrapped in its own try/catch (`Device '{Device}' poll cycle
      failed; continuing.`) so one bad gateway/DB hiccup can never stall the other
      gateways or kill the service. Sensor reads are split into two layers: MODBUS
      connect/read/parse failures drive the DC streak (unreachable meter), while DB
      `EnsureTableAsync`/`InsertAsync` failures are swallowed (sensor stays ONLINE,
      write retried next poll). Verified: 16/16 samples ONLINE, 0 flips.
    - **VERIFIED 100s monitor**: pingâ†’W610 always true, `dev=True sensor=ON` every
      sample, 0 flips, live values flowing. NOTE: effective telemetry cadence on this
      bridge is ~20-30s/update (serial bus is genuinely slow â€” each read is ~seconds);
      status stays ONLINE regardless. Powering the gateway OFF still flips it OFFLINE
      after the 60s watchdog.
14. **Live Monitoring = real SENSOR cards (2026-09-21)** â€” the user's "Live
    Monitoring" tab was showing FAKE legacy data (Boiler Feed Water, Chiller Line,
    Steam Line, "sdgsg"...) that was never added:
    - **Source was Firebase, not the API**: `LiveTelemetryState.LoadSnapshotAsync`
      called `ScadaDemoTestApiClient.GetLatestSnapshotAsync` which read the legacy
      Firebase RTDB (`FirebaseScadaService.GetDevicesAsync` + `GetRecentSensorReadingsAsync`)
      and painted FlowMeterCards. Removed the Firebase snapshot entirely â€” cards now
      come ONLY from the API SignalR hub (`/hubs/telemetry`).
    - **One card = ONE fully-added sensor** (user directive: "jab gateways/child me
      Add Sensor to Application se sensor aata hai tabhi dikhe; device cards nahi").
      `LiveReadingDto` gained `SensorExternalId`/`SensorName`; the poller
      `BroadcastSensorAsync(device, sensor, driver, ...)` stamps both columns.
      `LiveTelemetryState.MeterState` gained `SensorExternalId`/`SensorName` +
      a generic `Values` dict (metric column â†’ value+unit) so TemperatureC/HumidityRH
      AND InstantaneousFlowRate/AccumulatedTotalizer all paint; the old flow-only
      `TryMapMetric` filter was removed and Summary/Charts keep working off the
      device-level `FlowRate`/`Totalizer` convenience fields. Key = sensor id
      (fallback device external id). `Pages/Index.razor` now renders per-sensor
      cards (name + ONLINE/OFFLINE + every metric line + "via {gateway}") and the
      gateway-card fallback was removed entirely.
    - **Poller now broadcasts to SignalR**: `ModbusPollingHostedService` resolves
      `ITelemetryBroadcaster` and streams every successful sensor read + `IsOnline=false`
      on sensor-offline / watchdog / GATEWAY_OFFLINE transitions (fire-and-forget so a
      SignalR hiccup can never fail a poll). Alert status string no longer claims
      "Firebase Live".
    - **Gateway page Attached Sensor Children enriched**: each child row now shows
      sensor name, ID, slave (dec + hex), driver display, poll interval, calibration,
      live primary+secondary values + units + last-seen time, online badge.
    - Verified live (2026-09-21): US_TEST ONLINE, sensor S-A750 (Temp & Humidity
      Sensor (Slave 2)) ONLINE with live 45.2 Â°C streaming; both API :5080 and Web
      :5150 restarted with this build.
15. **Parameter selection is now ENFORCED end-to-end (2026-09-21)** â€” the user
    selects WHICH meter parameters a sensor reports at Add-time and can change that
    later ("jo select karo sirf wo DB me jaye, sirf wo show ho"):
    - **`Domain/Drivers/SensorColumnSet.cs`**: resolves the active telemetry columns
      from `Sensor.MetricFields` (driver-catalog names only). No selection (legacy)
      â†’ both driver columns; otherwise only the operator-selected ones. Handles the
      metadata-only `Velocity` param (never creates a column).
    - **`SensorTelemetryRepository`** is now column-aware: `CREATE TABLE IF NOT EXISTS`
      provisions ONLY the selected columns, plus idempotent `ALTER ... ADD COLUMN IF
      NOT EXISTS` for tables provisioned earlier with a different/full set. `INSERT`,
      `SELECT` (git both latest + recent) all reference only the active columns. Table
      provisioning in the poller is cached per (table + column signature) so an Edit
      re-runs the ADD COLUMN.
    - **Poller / Push ingest** writes and SignalR-broadcasts only the selected
      columns (`BroadcastSensorAsync` â†’ `SendLiveAsync` per active column) â€” an
      unselected parameter never reaches the DB insert nor the Monitoring card.
    - **API**: `PUT /api/sensors/{id}` (`UpdateSensorRequest`) edits name / poll
      interval / calibration / parameters. Server requires â‰¥1 parameter and â‰¥1
      MEASURABLE column (metadata-only selection â†’ 400). `SensorDto.MapAsync`
      surfaces `latestPrimary/latestSecondary` only for selected columns (unselected
      â†’ null â†’ "--").
    - **Gateway tab UI**: per-sensor "âš™ Edit Parameters" button opens an edit modal
      (pre-filled from current MetricFields) with Select-All/Clear, a count line, a
      "select at least one" warning, and Save via the new endpoint. The Add-Sensor
      config modal also gained Select-All/Clear + the â‰¥1 warning. `LiveTelemetryState`
      prunes stale metric entries (60s) so an un-selected metric's card line vanishes
      right after edit.
    - Verified live: sensor S-A750 (only TemperatureC selected) â†’ telemetry rows now
      carry `primaryValue=45.8, secondaryValue=0` (Humidity never written), API
      `latestSecondary=null`; PUT round-trip (add Humidity â†’ then back to Temperature
      only) works, 200 with `metricFields` updated and columns idempotently added.
      API :5080 / Web :5150 restarted (PIDs 3552 / 7800).
16. **30s cadence bug fixed + faster live data (2026-09-21)** â€” S-A750 updates were
    arriving every EXACTLY ~30s despite PollIntervalSeconds=5:
    - **Root cause**: `IsBackingOff` used `_lastAttemptUtc`, which was stamped on
      EVERY poll attempt (line "`_lastAttemptUtc[device.Id] = DateTime.UtcNow;`"
      before the sensor loop). `GatewayBackoff` = 30s, so after ANY successful poll
      the next 30 seconds every cycle hit `IsBackingOff()` == true and `continue`d
      â†’ exactly one poll per 30s regardless of poll interval.
    - **Fix**: backoff is now set ONLY when the device is actually unreachable
      (`else if (!lastSeenFresh)` â†’ mark offline branch) and REMOVED on success
      (`anyConnected` â†’ `_lastAttemptUtc.Remove(device.Id)`). A healthy gateway is
      polled every cycle per its sensors' poll intervals again. Verified: cadence
      30s â†’ ~6s (poll 5s + 1s tick), then after PUT poll interval 5sâ†’2s â†’ ~3s,
      values streaming constantly, device stays ONLINE, spurious 0/garbage rows gone.
    - Bridge reads measured ~100ms each (direct FC03 probe 77-177ms) â€” the W610 is
      NOT slow; the poller was simply gating itself.
    - **Poll interval**: sensor S-A750 now 2s (user wants near-real-time). `IsDue`
      floor stays `Math.Max(1, PollIntervalSeconds)`.
    - **UI cleanup (user request)**: removed the "Poll every X s â€¢ Cal Ã—Y â€¢
      telemetry_â€¦" detail line from Gateway "Attached Sensor Children" rows, and
      removed the "Selected âœ“" text next to each gateway card's ðŸ” Find Sensors
      button. Rows now show name / ID / slave hex / driver / live values + last-seen.
    - API :5080 + Web :5150 restarted (killed before build due to exe lock).
10. **Gateway tab workflow (IMPLEMENTED 2026-09-21)** â€” the Smart-Scan work moved
    OUT of the Norvi/Devices page into a brand-new **Gateway** page; Devices was
    restored to its pre-scan state:
    - **Nav**: `AppTabs.Gateway` added (auto-policies `Tab:Gateway` on API + Web).
      `NavMenu.razor` rewritten: a "Devices" parent expander with children
      **Norvi** (`/devices`) and **Gateway** (`/gateway`); CSS `.nav-group-*`/
      `.nav-child`/`.nav-caret` in `site.css`.
    - **`Devices.razor` REWERTED**: Smart Scan button, results panel, scan state
      fields (`ScanResults`/`ScanningDeviceIds`/`ScanFailedDeviceIds`) and methods
      (`ScanDeviceBusAsync`/`AddFromScanAsync`/`ClearScan`/`MeterReading`) removed â€”
      it's back to the plain Devices page.
    - **`Pages/Gateway.razor` (NEW)**: full workflow
      Step 1 Add Gateway (modal, USR-W610 default, IP/port/baud/parity/timeout +
      duplicate-IP guard) â†’ Step 2 Find Sensors (`ScanGatewayBusAsync`,
      probe 20 ms, registered slaves auto-skipped) â†’ Step 3 detection list with
      multi-select checkboxes + Select All (unknown responders disabled) â†’
      "Confirm & Add to System" â†’ Step 4 per-meter card â†’ "Configure & Add" modal
      (editable name, editable slave id pre-filled + duplicate-in-bus checker,
      poll interval, calibration, and a **parameter multi-select** from
      `ParamCatalog` per driver) â†’ "Add Sensor to Application" â†’ sensor created as
      a child; gateway card shows an "Attached Sensor Children" tree with
      ONLINE/OFFLINE badges. Selected gateway, live children and gateway status
      auto-refresh every 4 s. ONLINE/OFFLINE = real reachability semantics.
    - **Param persistence**: API `SensorsController.CreateSensorRequest` gained
      optional `SelectedParameters` (Dictionary) + `SensorConfig`;
      `Sensor.MetricFields` = JSON of the selected params (key â†’ unit) and
      `Sensor.Config` stored; `SensorDto`/`MapAsync` expose both. Web
      `CreateSensorRequest`/`AttachedSensorDto` updated to match.
    - **`ParamCatalog`** (in `Gateway.razor`) mirrors `ModbusMeterLibs`: AOSONG â†’
      TemperatureC Â°C / HumidityRH %RH; KAIFENG_FLOWMETER â†’ FlowRate NmÂ³/h /
      Totalizer NmÂ³ / Velocity m/s (metadata-only); KAIFENG_EM_FLOWMETER â†’ mÂ³/h / mÂ³.
    - `ScanGatewayBusAsync` in the client now parses the 502 body too â†’ partial
      scan results (ConnectivityOk=false) are shown alongside the unreachable
      warning. Built API + Web clean (0 warnings/0 errors).
    - **NOTE**: **API (:5080) + Web (:5150) were RESTARTED 2026-09-21 with the
      Gateway-tab build** (detached WMI starts â€” verify with
      `Get-NetTCPConnection -LocalPort 5080,5150`; latest PIDs API 7740 / Web 6852).
      Verified: login `superadmin@alamiot.com` / `SuperAdmin@123` works,
      `/api/devices` 200 (1 device), `/api/sensors/libraries` 200 (3 drivers),
      Web `/`, `/devices`, `/gateway` all 302 â†’ `/account/login` (normal for
      unauthenticated Blazor Server). BUGFIX (2026-09-21): scan 502 body is
      `{message, scan}` â†’ client unwraps it (`ScanBusErrorDto`) so a gateway
      unreachable no longer NRE'd (`Found` null â†’ Crash at line 119). `FindSensorsAsync`
      normalizes `Found`/`SkippedSlaves` to empty lists; scan-area button is now a
      always-visible **"+ Add Sensor to Application"** (disabled until â‰¥1 identified
      meter selected) that directly opens the parameter-configure modal; after a
      successful add it auto-advances to the next selected meter.
17. **Charts/Reports/Alerts moved to REAL API telemetry (2026-09-21)** â€” all three
    legacy pages ran on Firebase RTDB demo data / client-side fabricated series:
    - **`Pages/Charts.razor` FULL REWRITE**: Firebase readings + Storage Tanks +
      hash-fabricated Trend Comparison removed. Now one live chart card per real
      sensor (from `GetGatewayDevicesAsync` + `GetSensorsAsync`), streaming
      `/api/sensors/{id}/telemetry?count=60` on a 1s timer via `drawLineChart`
      (chronological order). "Trend Comparison" tab = two real sensors overlaid on
      separate canvases (sensor pickers). No leftover demo paths. Tab label in nav
      renamed "Charts & Trends" (no tanks anymore).
    - **`Pages/Reports.razor` FULL REWRITE**: per-sensor date-range builder
      (sensor + Primary/Secondary metric + local datetime range + presets) querying
      the per-sensor telemetry table, with avg/max/min/online stats + raw-samples
      table. CSV export downloads client-side via new `downloadTextFile` JS helper
      (blob; Blazor Server can't `window.open` the API URL â€” needs the Bearer token).
      API `GET /api/sensors/{id}/telemetry` gained optional `from`/`to` (UTC, clamped
      1..2000) + `GET /api/sensors/{id}/telemetry/export-csv`. New repo method
      `SensorTelemetryRepository.GetRangeAsync` + `QueryAsync(from,to)` (postgres
      `@from IS NULL OR "TimestampUTC" >= @from` pattern).
    - **`Pages/Alerts.razor` + Web client Alerts methods**: moved OFF Firebase onto
      the existing Postgres `AlertsController` (`/api/alerts/incidents|rules`,
      acknowledge, create/delete rule â€” already present but unused by Web). Rule
      builder now targets **sensors** (`UniqueSensorId` or ALL) with REAL driver
      metric keys (InstantaneousFlowRate, AccumulatedTotalizer, TemperatureC,
      HumidityRH) instead of fake FlowRate(L/min)/Pressure. "Notification Channels
      Email+Push Active" fake card removed.
    - **Alert rule engine now LIVE**: `AlertService.EvaluateReadingAsync`
      (Infrastructure/RealTime, was dead code) is called from
      `ModbusPollingHostedService` after every successful sensor read for each
      SELECTED column (`deviceExternalId = sensor.UniqueSensorId`, `deviceName =
      sensor.Name`). Threshold violation â†’ `AlertIncidents` row (2-min debounce) +
      SignalR `ReceiveAlertIncident` broadcast. Alert hiccup is try/caught â€” never
      fails a poll.
    - **Rollup verified**: `RollupCompressionHostedService` compresses the legacy
      `SensorReadings` table, which NOTHING writes anymore (poller writes per-sensor
      `telemetry_sensor_*` tables) â†’ it silently no-ops each tier. Raw rows stay
      forever in the per-sensor tables. Left as-is (harmless); retarget later if the
      per-sensor tables need compression.
18. **Poll health: reconnect-retry gated (2026-09-21)** â€” the USR-W610 serial bridge
    (~3s/read, erratic; 50%+ timeout at 8s) made failures expensive: the one-shot
    reconnect retry doubled every failed read to ~10s, racing the 3-strike offline
    streak (EM sensor flapped OFFLINE twice). `ModbusPollingHostedService` now retries
    ONLY when the first attempt failed FAST (`ElapsedMilliseconds < ReadBudgetMs/2`);
    a genuine read timeout skips the retry (it would just time out again). Cost of a
    slow-bridge miss â‰ˆ 1 read budget, not 2.
19. **Nav order + Reports rendering fix (2026-09-21)**: Devices parent group moved ABOVE
    Reports (order: Live Monitoring, Summary, Charts & Trends, **Storage Tanks**, **Devices (Norvi/Gateway)**,
    Reports, Alerts, ...). `NavMenu.razor` now uses ONE ordered `AllNavEntries`
    (List<object> of NavItem/NavGroup) instead of separate item+group loops. ALSO fixed
    a real bug the user hit in Reports: a PowerShell `Set-Content`-based edit in an
    earlier session had double-encoded every non-ASCII char (â³â†’"Ã¢Â³", Â°â†’"Ã‚Â°", â€”â†’"Ã¢â‚¬"â€”)
    in Reports.razor, so the page rendered mojibake. Rewrote the file clean UTF-8 and
    simplified headings/labels (no emoji clutter in Reports).
    RESTARTED: API :5080 (PID 12876) + Web :5150 (PID 14536), both build 0 warnings.
20. **LIVE verification sweep (2026-09-22)** â€” API + Web restarted (build 0 warnings/0
    errors), all fallbacks curl-verified:
    - Login `superadmin@alamiot.com` OK; `/api/devices` â†’ 1 device `My_TEST`
      (`DEV-C4818E0E`, USR-W610) with 2 sensors: AOSONG **S-6348** (Slave 3,
      `telemetry_sensor_aosong_726de6a8`) + Kaifeng EM **S-F64F** (Slave 1,
      `telemetry_sensor_kaifeng_em_8ef4a2e7`). Both OFFLINE â€” the W610 @
      10.10.100.254/502 is currently UNREACHABLE from this machine (hardware/power),
      so OFFLINE is the CORRECT data-driven state.
    - **Poller alive**: wrote exactly ONE watchdog offline row today 04:27 UTC
      (onlineâ†’offline transition) â€” new build (alert wiring + retry gate) runs fine.
      Yesterday's live rows retained: 60-77 Â°C / 32 %RH in the AOSONG table.
    - **Reports range**: `GET /api/sensors/{id}/telemetry?from=&to=` â†’ 93 rows;
      `export-csv` â†’ 200, 3154 bytes, correct header + rows. Secondary all 0/absent
      (only TemperatureC selected â€” column-aware writes confirmed).
    - **Alerts round-trip**: create rule (S-6348 TemperatureC >45 Warning) â†’ list (5) â†’
      DELETE all â†’ 0 rules; acknowledge endpoint resolved the 2 dummy incidents.
      CLEANED UP legacy fake-era rules (FlowRate/Pressure on ALL â€” metrics that never
      match driver columns) + dummy incidents left by earlier manual testing. The
      live ruleâ†’incident trigger fires the moment the W610 returns (poller already
      calls `AlertService.EvaluateReadingAsync` per selected column on success).
    - Web `/`, `/reports`, `/charts`, `/alerts`, `/devices`, `/gateway` â†’ 302
      `/account/login` (normal unauthenticated Blazor Server).
21. **UX cleanup round (2026-09-22)** â€” user closing-out batch: removed dead tabs,
    faster/reliable modal toasts, real-time alert toast, PDF export back in Reports,
    Charts overhaul with time axis + level bands + KPI strip, formal login:
    - **Tabs removed**: `System Settings` (Settings.razor â€” was full of fake/sim) and
      `Firmware OTA` (FirmwareOTA.razor + API `FirmwareController.cs`) deleted from
      nav and disk. Firmware code ARCHIVED (out of compile): `src/FirmwareOta_Archived/FirmwareController.cs.archive.txt`
      + `FirmwareOTA.WebPage.razor.archive.txt`. Nav order unchanged otherwise.
      `AppTabs.FirmwareOTA`/`Settings` constants + Web client firmware methods remain
      (unused, harmless). Firmware release/rollout API endpoints are GONE from the API.
    - **Faster, reliable modal success**: `ToastAsync` (Devices/Gateway) and `ShowToast`
      (Alerts) now auto-dismiss after **1700 ms** with a `_toastSeq` token (a newer
      toast can't be erased by an older timer). Devices/Gateway modal success delay
      800â†’450ms. Alerts `ShowToast` kept `void` (fire-and-forget Task.Run) so the 10
      call sites didn't need `await`. Reports `ShowToast` is async + awaited.
    - **Alerts notify IN-APP now**: alert rule engine (already live in the poller)
      broadcasts `ReceiveAlertIncident`; Web `LiveTelemetryState` subscribes it and
      raises `OnAlertIncident`; `MainLayout.razor` renders a top-right toast stack
      (severity-colored Info/Warning/Critical, auto-dismiss 6s). **Alert rule ENGINE
      matches driver-column metrics only** â€” see the user-rules gotcha in Known context.
    - **Reports PDF back**: new `GET /api/sensors/{id:guid}/telemetry/export-pdf?from=&to=`
      (SensorsController, QuestPDF, `[Authorize Action:DevicesView]`): header/stats cards
      (count, avg, min/max, online %) + up to 250 data rows + footer. Web Reports gained
      **Export PDF** button; `ScadaDemoTestApiClient.GetSensorTelemetryPdfAsync` returns
      bytes; `charts.js` gained `downloadBlob(filename, base64, mime)` (binary).
      VERIFIED: 200, `application/pdf`, 59 KB real PDF.
    - **Charts.razor overhauled**: window chips (Last 1 Hour / 6 Hours / 1 Day / 7 Days â†’
      recent-count vs bounded `from/to` range, max count 1500); every canvas now shows
      **time axis labels** (window start â†’ latest sample, `HH:mm` or `MM-dd HH:mm`);
      per-sensor **LOW/MEDIUM/HIGH level chip** from live value + unit bands (Â°C â‰¤20/<60 / %RH â‰¤25/<70 /
      flow â‰¤0.1/<100 / mÂ³ â‰¤10/<10000); **Plant Snapshot KPI strip** (Sensors online,
      Temperature, Humidity, Flow Rate, Totalizer from real latest readings â€” no fake
      tanks); draw skipped unless the latest reading changed (per-sensor signature cache);
      sensor metadata reload ~6s, charts redraw 2s; comparison tab unchanged but shares
      the range labels. Empty state text only (no emoji).
    - **Monitoring (Index.razor)**: "System Telemetry: connectingâ€¦" widget REMOVED;
      subtitle now `.page-subtitle-soft` (lighter); the "via {gateway} / Last seen"
      footer is a dedicated `.meter-conn` block (status dot pill + two-line muted text,
      dashed top border) so the connect line reads clearly against the card.
    - **Summary**: real-data **KPI strip** (Sensors / Online / Offline / Live Link) +
      full `yyyy-MM-dd HH:mm:ss` heartbeat timestamps.
    - **Login formal**: password input now empty by default (no pre-filled secret),
      SVG eye/eye-off toggle replaces the emoji, SVG alert icon in the error banner,
      `.eye-icon`/`.error-icon` sizing in site.css.
    - JWT maintained everywhere touched: Alerts GETs (`incidents`,`rules`) got
      `[Authorize(Action:AlertsView)]`; PDF endpoint authorized; new pages use the
      existing forwarded-bearer client.
    - VERIFIED live (this round): both builds **0 warnings / 0 errors**; loginâ†’token;
      `/api/devices` 1 device; `/api/alerts/rules` â†’ **3 rules the USER created today**
      (see Known context gotcha); `/api/sensors/{id}/telemetry` range 200; export-csv
      200 (3240 B); export-pdf 200 (59 KB); libraries 3 drivers; all Web routes render.
      RESTARTED: API :5080 (listener PID 12504) + Web :5150 (listener PID 13768).
22. **Storage Tanks + compact monitors + real time-axis charts (2026-09-22)** â€” user
    batch: smaller monitoring cards, real time-of-day x-axis inside every chart
    (not just outer chips), and a literal liquid-tank dashboard modeled on the
    user's pasted "AQUA" HTML design â€” all real-data, zero simulation:
    - **Smaller monitoring cards**: `.meter-grid` minmax 340â†’300px + gap 24â†’18;
      `.meter-card` padding 24â†’16, gap 16â†’11; `.meter-value` 2.1â†’1.7rem;
      `.meter-name`/`.meter-unit`/`.meter-label`/`.btn-report` scaled down â€”
      more cards per row, same Online/Offline + via-gateway + Last-seen line.
    - **Real time axis on charts**: `charts.js` `drawLineChart` gained an 8th arg
      `times` (epoch-ms array aligned 1:1 with values). Chart x-positions are now
      proportional to REAL timestamps; 5 tick labels (`HH:mm`, or `MM-dd HH:mm`
      when the window span > 24h) render under each canvas with vertical
      gridlines. Charts.razor passes `ordered.Select(...ToUnixTimeMilliseconds())`
      on both the per-sensor charts and the Trend Comparison canvases (UTC ms â†’
      browser-local time-of-day). Backwards-compatible (old 7-arg calls still work).
    - **Tanks = opt-in flag on a sensor** (no new DB table, no simulation): a sensor
      becomes a tank ONLY when its `Config` JSON says so â€”
      `{"isTank":true,"capacity":<mÂ³>,"liquidType":"Water","levelSource":"AccumulatedTotalizer"}`.
      The API already round-trips `SensorConfig`/`sensor.Config` via the existing
      Create/Update DTOs â€” zero API changes, verified live end-to-end (temp
      sensor S-C618: created with config â†’ GET echoed it â†’ deleted; device kept
      only S-6348 + S-F64F).
    - **"Storage Tank asset" toggle** (dashed blue box + capacity + "Holds") added
      to the Gateway **Configure & Add** modal AND the Gateway **Edit Parameters**
      modal AND the Devices/Norvi **Add Field Sensor** modal. Shown ONLY for
      flowmeter drivers (`KAIFENG_FLOWMETER`, `KAIFENG_EM_FLOWMETER`) â€” temp/humidity
      sensors can't be tanks (no volume totalizer). Edit-modal unticking sends `""`
      so the flag is really cleared (API: `Config = req.SensorConfig ?? Config`).
    - **`Pages/Tanks.razor` (NEW, `/tanks`, `[Authorize(Policy = "Tab:Tanks")]`,
      nav "Storage Tanks" right after Charts & Trends)**: light-theme AQUA-style
      SVG tank cards â€” dashed level marks 0/25/50/75/100, rounded shell, animated
      sine wave (CSS `tkWave` keyframes, translate -60px = one full wave period,
      two staggered layers), glass shine, FULL/EMPTY overlay tags (HTML overlay â€”
      SVG `<text>` collides with Razor), big % readout, Volume/Flow/Capacity/Holds
      stats, ONLINE/OFFLINE pill, `S-XXXX â€¢ Slave n â€¢ via gateway`, full
      `yyyy-MM-dd HH:mm:ss` last-seen. Level% = live `AccumulatedTotalizer` Ã·
      capacity (clamped 0-100). Live values come from the shared
      `LiveTelemetryState` SignalR bus keyed by `UniqueSensorId` (flow/vol only
      for columns the operator selected); sensor/config list reloads every 4s;
      `OnChange` re-renders instantly. Offline/never-seen tank â†’ `--`% with muted
      styling, no crash. Empty state explains how to flag a tank.
    - **Tank CSS** in `site.css`: `.tk-grid/.tk-card/.tk-svg-wrap/.tk-svg-tag`,
      `.tk-read/.tk-pct/.tk-stat*`, `.tk-conn*` (mirrors `.meter-conn`),
      `.tk-wave-a/.tk-wave-b` + `@keyframes tkWave`, `.tank-toggle-box/-title/
      -hint`, `.tank-checkbox`.
    - Builds: API + Web **0 warnings / 0 errors**. Fixed during round: SVG `<text>`
      â†’ RZ1023 (Blazor `<text>` tag clash â€” replaced with HTML overlay); CS0542
      `Tanks` field = component class name (renamed `TankList`); CS0219 unused
      const; exe-copy MSB3026 was only the running-process file lock.
    - VERIFIED live (new PIDs): login (`email` field â€” the DTO is
      `LoginRequestDto.Email`, not `username`) â†’ token; `/api/devices` 200 (1
      device, 2 sensors); `/api/sensors/libraries` 200 (3 drivers);
      `/api/alerts/rules` 200 (still the 3 stale user rules â€” see gotcha);
      sensor telemetry range + export-csv + **export-pdf all 200**; tank
      Config createâ†’readâ†’delete round-trip 200/200/200; Web `/` `/tanks`
      `/gateway` `/charts` `/devices` `/reports` `/alerts` all 302 â†’ login
      (routes registered; `/tanks` is NOT a 404). RESTARTED: API :5080
      (listener PID 2176) + Web :5150 (listener PID 11112).
23. **Dashboard tab + TREND group = Charts/Tanks only (2026-09-22)** â€” user correction
    round: the arc-gauge plant overview is now the TOP-LEVEL **Dashboard** tab, and
    TREND only holds Charts + Storage Tanks. Live Monitoring and Totalizer Summary
    sit on the top level again (NOT inside TREND):
    - **`Pages/Energy.razor`** = the Dashboard: `@page "/dashboard"` + alias
      `@page "/energy"`, `[Authorize Policy="Tab:Energy"]` (AppTabs.Energy in `All`
      â†’ Web `Tab:Energy` policy auto-generated). **One full-page overview of the
      whole system, all real data**: KPI strip (Sensors total / Online / Offline /
      Gateways online / Total flow Î£ / Total volume Î£ / Active alerts count), a
      **Gateways section** (card per gateway: name + external id + ONLINE/OFFLINE
      pill + hardware type, IP:port, sensors count/max, last seen), the **Live
      Sensors section** â€” a 2-col grid of **one arc-gauge group card per registered
      sensor** (auto â€” add/sensor appears in â‰¤4s, nothing to configure): navy header
      (name + `S-XXXX` tag + ONLINE/OFFLINE pill) â†’ two halves, each with a **260Â°
      arc gauge** (raw-SVG as `MarkupString` so `<text>` is allowed, unique per-metric
      gradient, 0/25/50/75/100% ticks, animated stroke-dashoffset) + rows (Live value /
      Unit / Scale 0â€“max). Footer: Slave n â€¢ via gateway + last-seen. Bands by unit:
      Â°C/%RH â†’ 100, mÂ³/h/NmÂ³/h flow â†’ 1000, mÂ³ volume â†’ 10000. Primary/secondary =
      operator's selected MetricFields (falls back to DTO latest columns). Values
      stream from `LiveTelemetryState` SignalR (SyncLiveValues on `OnChange` + 4s
      timer + Refresh button), fall back to DTO late values â€” and a **Recent Alerts
      section** (last 15 `AlertIncidents`: severity pill Info/Warning/Critical, message,
      metric = value, device, time, Acknowledge button per unacked incident that calls
      `Api.AcknowledgeAlertAsync` + reloads incidents).
      All-offline notice `.eg-warn` (amber strip: "registered but none online - gateway
      not reachable") appears when online count == 0 so it never looks broken/empty.
      KPI sums only use ONLINE live values; `--` when none.
    - **`NavMenu.razor` order**: **Dashboard `/dashboard`** (top-level, bolt icon,
      `Tab:Energy`) â†’ **TREND group** (children **Plant Energy `/energy`** â€” the
      reference "PLANT UTILITIES / Category Wise" design below, bolt icon, SAME
      `Tab:Energy` policy but a **separate page** + **auto-one-card-per-sensor**
      just like `/dashboard`; + Charts & Trends `/charts` + Storage Tanks `/tanks`)
      â†’ **Live Monitoring `/`** â†’ **Totalizer Summary `/summary`** â†’ Devices group
      (Norvi/Gateway) â†’ Reports, Alerts, Multi-Site, Audit Logs, Users.
      `OpenGroups` defaults `{"TREND", AppTabs.Devices}` so TREND expands on load.
      New icons `TrendsIcon` (activity) + `EnergyIcon` (bolt). NOTE: the energy
      dashboard is a TREND CHILD (user directive 2026-09-22: "dashboard TREND se
      bar nikal lo, or ye jo code diya PLANT UTILITIES wala us ka TREND me ek alag
      tab banao, readings real ho"). `/dashboard` and `/energy` are DIFFERENT pages.
      **PlantEnergy.razor gauge animation**: each gauge emits stable ids
      `pe-fill-{id}` / `pe-val-{id}` (id = `pe` + first-8 hex of sensorId + metric
      index); `TrySet` updates `Frac/DisplayValue` + sets `_animDirty`; after every
      render `OnAfterRenderAsync` sends one `animPeGauge(id,offset,value)` batch to
      `charts.js` (`window.animPeGauge`), so the CSS `stroke-dashoffset .9s
      cubic-bezier` transition animates the needle smoothly per live reading â€”
      no SVG rebuild/jump. Gradients `egrad-{id}` navyâ†’blue2 per gauge.
    - **CSS** (`site.css`): `.eg-toolbar/.eg-muted/.eg-warn/.eg-clock`,
      `.dash-section-title/.dash-gateways/.dash-gw*/.dash-alerts/.dash-alert*/
      .sev-pill[.warning|.critical|.info]`, `.eg-groups/.eg-group`
      (2-col, hover lift), `.eg-head` (brand gradient #0369a1â†’primary-blue),
      `.eg-title/.eg-tag/.eg-headright`, `.eg-body/.eg-half/-empty`,
      `.eg-gauge[.offline]` (grayscale), `.eg-arc-bg/.eg-arc-fill/.eg-tick-lb/
      .eg-g-val`, `.eg-rows/.eg-row`, `.eg-foot/.eg-foot-dot/.eg-conn-*`.
      Responsive: 1-col <1080px, halves stack <560px. PlantEnergy `pe-*` block:
      `.pe-topbar/-in/-brand/-badge/-title/-t1/-t2/-actions`, `.pe-wrap/-toolbar`,
      `.pe-kpis/.pe-kpi`, `.pe-groups/.pe-group[.pe-offline]/.pe-g-head/.pe-tag`,
      `.pe-g-body/.pe-half/.pe-gauge[.offline]`, `.pe-arc-bg/.pe-arc-fill
      (transition .9s)/.pe-tick-lb/.pe-g-val`, `.pe-rows/.pe-row/.pe-empty`,
      `.pe-foot/.pe-footline/.pe-note/.pe-empty-state`; responsive mirrors the
      reference (4â†’2â†’1 col KPIs, groups 2â†’1, halves stack <560px).
    - **Gateway page delete (2026-09-22)** `Pages/Gateway.razor`: each gateway card
      now has a red **Delete** button + each "Attached Sensor Children" row a
      **Delete Sensor** button (red inline style `rgba(239,68,68,0.1)`, same as
      Devices page). Both open a confirm modal (`DeviceToDelete`/`SensorToDelete`
      + `DeleteBusy`), call `Api.DeleteGatewayDeviceAsync`/`Api.DeleteSensorAsync`
      (existing DELETE endpoints), clear selection if the deleted device was
      selected, reload via `LoadGatewaysAsync`/`RefreshChildrenAsync`, toast on
      result. Build 0 warnings/0 errors.
    - **Gateway children always visible (2026-09-22 BUGFIX)**: "Attached Sensor
      Children" used to render ONLY inside `@if (isSelected)` â€” a freshly-added sensor
      was invisible until the user clicked the gateway card, so delete/edit seemed
      missing. Now each gateway card ALWAYS shows its child-sensor rows (name/ID/slave/
      driver/live values/online badge + Edit Parameters + Delete Sensor buttons) via a
      new `Dictionary<Guid,List<AttachedSensorDto>> AllChildren` populated in
      `LoadGatewaysAsync` (`GetSensorsAsync()` no-arg â†’ group by `DeviceId`) and
      refreshed per-device in `RefreshChildrenAsync` + on sensor delete/device delete.
      Empty gateways show "No field sensors attachedâ€¦" hint. Build 0 warnings/0 errors.
    - Builds: API + Web **0 warnings / 0 errors** (fixed: CS8602 `meter` out-var
      null-flow in SyncLiveValues â€” now `TryGetValue(...) && meter is not null`;
      removed undefined `var(--inset)` from `.eg-arc-bg`; PlantEnergy CS0219 unused
      `gid` const removed). VERIFIED live: Web
      `/dashboard` `/energy` `/charts` `/tanks` `/summary` `/` `/gateway`
      `/devices` â†’ 302 (all registered), API `/api/devices` â†’ **1 gateway My_TEST
      ONLINE with 2 sensors**: `S-6348` (AOSONG, slave 3, 45.8 Â°C / 30.5 %RH) +
      `S-F64F` (Kaifeng EM, slave 1, 0 mÂ³/h / 0.57 mÂ³) â€” both ONLINE, W610
      reachable again (last readings 08:17 UTC).
      RESTARTED: API :5080 (listener PID 3960) + Web :5150 (listener PID 17924).
    - **Dashboard/Energy offline-mislabel BUGFIX (2026-09-22)**: `SyncLiveValues` on
      the Dashboard (`Energy.razor`), Plant Energy (`PlantEnergy.razor`) and Storage
      Tanks (`Tanks.razor`) used to derive card `IsOnline` ONLY from the SignalR live
      bus (`State.Meters`) and stamped `false` when the bus had no reading for that
      sensor â€” so a perfectly ONLINE sensor (API DTO `isOnline=true`, fresh
      timestamps) rendered OFFLINE whenever the browser just-loaded/hub was silent.
      Now the base online state = the API DTO's worker-maintained `IsOnline` (real
      poll/push success); the live bus only OVERRIDES it when it actually has the
      reading (overrides with fresh LastSeen too). Applies to Kpis/counts as well.
    - **Dashboard/Energy color boost (2026-09-22)**: `.eg-*`/`.pe-*` gauges looked
      washed-out. Bolder strokes: `.eg-arc-bg`/`.pe-arc-bg` track #dbe3ec â†’ #c3d2e3/
      #c6d6ea; `stroke-dashoffset` drop-shadow glow; gauge value text #1e3a5f/#16345a
      weight 800 size 16; tick labels weight 700; card headers deeper gradient
      #075985â†’#0389d0; PlantEnergy `argsNavy/argsBlue2` â†’ #075985/#38bdf8.
24. **UX batch #2 (2026-09-22)** â€” nav order + alerts + mailer + titles + cards:
    - **Nav order** (`NavMenu.razor`): Dashboard top-level â†’ Live Monitoring `/` â†’
      Totalizer Summary `/summary` â†’ **TREND group** (Plant Energy `/energy`, Charts
      `/charts`, Storage Tanks `/tanks`) â†’ **Devices group** (Norvi `/devices`,
      Gateway `/gateway`) â†’ Reports `/reports` â†’ Alerts `/alerts` â†’ Multi-Site â†’
      Audit Logs â†’ Users. `OpenGroups` still `{"TREND", AppTabs.Devices}`.
    - **Alerts** (`Alerts.razor` + `AlertsController` + `ScadaDemoTestApiClient`):
      - Every incident row now has a ðŸ—‘ï¸ delete button â†’ new `DELETE
        /api/alerts/incidents/{id}` (`AlertsView` policy, audit-logged). Admin
        resolved rows too â€” hard-removes the incident, not just acknowledge.
      - Rule modal **Metric dropdown is driver-aware**: selecting a specific sensor
        filters metrics to that driver's real columns (AOSONG â†’ TemperatureC/
        HumidityRH only; Kaifeng/EM â†’ InstantaneousFlowRate/AccumulatedTotalizer
        only); `ALL Sensors` keeps all 4. `OnRuleTargetChanged` auto-picks first
        metric. Server-side metrics were already real driver columns â€” this fixes
        the UI offering irrelevant metrics for a sensor.
    - **Email mailer wired** (`AlertService` + `appsettings.json`):
      `AlertService.EvaluateReadingAsync` now also calls `SendEmailAsync(rule,
      incident, ct)` after a NEW incident is persisted â†’ SMTP via hosted mailer.
      `Smtp` config section: Host smtp.gmail.com, Port 587, EnableSsl true,
      Username/From muhammad.hamzx.787@gmail.com, Password = Gmail app password
      **`bapz plwd ycpf pjor`** (WITH spaces â€” plain no-space form 530s, Gmail
      accepts the spaced 16-char app password; verified live: SMTP send OK 2026-09-22).
      Recipient = rule.NotificationEmail (fallback Smtp:DefaultRecipient).
      Injects IConfiguration; email failures are try/caught + logged only.
    - **Dashboard title** (`Energy.razor`): `<h1>` simplified to just **Dashboard**
      (was "Dashboard â€” Complete Plant Overview").
    - **Plant Energy header** (`PlantEnergy.razor`): "ENERGY â€¦ MANAGEMENT" brand
      text and "Dashboard Summary â€” Category Wise" removed/replaced â€” topbar now
      shows only the âš¡ badge + centered **Plant Energy** (matches tab label).
    - **Live Monitoring boosts** (`Index.razor` layout via `site.css`):
      `.meter-grid` minmax 300pxâ†’260px, gap 18â†’16 â†’ up to 3 cards per row on a
      typical 1080-1280px-wide browser (user wanted 3-across, not 2-wide holes).
      Card auto-connect is SignalR-driven (fast); no forced page refresh needed.
    - **Reports raw samples bounded** (`Reports.razor`): table wrapper now
      `max-height:420px; overflow-y:auto` (no more full-page 221-row scroll) and a
      **Load next 200** button pagination (`SamplesShown` starts 200, reset on each
      report run) replaces the old "Showing first 200 of N..." dead-end row.
    - Builds: API + Web **0 warnings / 0 errors** (fixed: CS8604 `from!` null
      null-annot in MAIL; CS1525 nested-quote in `@onchange` â†’ moved null-handling
      into `OnRuleTargetChanged(string?)`). VERIFIED: login token OK, all Web
      routes 302, incidents+rules endpoints 200, `DELETE incidents/{fake} â†’ 404`
      (route + auth registered).
      RESTARTED: API :5080 (listener PID 2388) + Web :5150 (listener PID 6492).
25. **DATABASE SWITCH: Supabase â†’ local SQL Server LocalDB (2026-09-22)** â€” the
    whole platform now runs on a local empty database `scada_db_iiot` at
    `(localdb)\MSSQLLocalDB`. Supabase/Postgres code is COMMENTED OUT (not
    deleted) â€” full revert path + change log in `DB_LOCAL_SWITCH_CHANGELOG.md`,
    old Postgres migrations at `backup_postgres_migrations/`:
    - **csproj**: `Npgsql.EntityFrameworkCore.PostgreSQL` commented;
      `Microsoft.EntityFrameworkCore.SqlServer` 10.0.11 added.
    - **appsettings.json**: LocalDB connection string active; Supabase string
      kept as comment (`//` is tolerated by .NET config loader).
    - **Program.cs**: `UseSqlServer` (was `UseNpgsql`); startup baseline SQL
      rewritten to T-SQL (`sys.tables`/`OBJECT_ID(N'__EFMigrationsHistory')`
      guard, dashed SQL insert matching the old other-provider history stamp).
      **FIXED later same day**: baseline's legacy-history stamping (inserted
      `InitialPostgres` into `__EFMigrationsHistory` whenever `AlertIncidents`
      existed â€” i.e. every boot after first) removed entirely. `MigrateAsync`
      alone now handles fresh vs already-migrated DBs; history stays clean.
    - **EfResilience**: `SqlException` (Microsoft.Data.SqlClient) instead of
      NpgsqlException; `IsTransient` handles network/constraint/timeout.
      **LOCAL TUNING (later same day, user's "reading delay" fix)**: Supabase-era
      retry policy (4Ã— exponential 200ms*2^n + jitter â†’ up to ~3s stall, sized for
      a REMOTE cloud DB) replaced by **3 quick retries at fixed 100ms** â€” the old
      backoff stalled the poller's sequential per-sensor insert loop and pushed
      back the other sensors on the same gateway. No other Supabase-era delay
      existed (grep over appsettings = 0 hits; poller/watchdog/budget constants
      are hardware/serial-bridge tuning, untouched). Polling still never fails â€”
      transient SQL errors are still retried. Details in `DB_LOCAL_SWITCH_CHANGELOG.md`.
    - **SensorTelemetryRepository**: full T-SQL rewrite â€” `SqlParameter`,
      `OBJECT_ID`/`COL_LENGTH` guards, `bigint IDENTITY(1,1)`, `datetime2`,
      `float`, `nvarchar(max)`, `smallint`, `SYSUTCDATETIME()`, `TOP (@count)`,
      `sys.tables`/`sys.columns` for existence; GetRange filter uses
      `@from/@to` with `IS NULL OR` bounds â€” provider-agnostic insert/query.
    - **DynamicSchemaInitializer**: T-SQL DDL rewrite â€” `uniqueidentifier`/
      `bit`/`datetime2`/`float`/`nvarchar(max)`, guarded CREATE/ALTER with
      `IF COL_LENGTH(...) IS NULL`, `sys.indexes`/`sys.foreign_keys` guards.
    - **Backup folder moved OUT of the csproj tree**: `Migrations_Postgres_Backup`
      â†’ repo-root `backup_postgres_migrations/` (Npgsql files cannot compile
      without the commented-out package).
    - **Fresh migration**: `20260922112111_InitialSqlServer` generated with
      `dotnet ef migrations add InitialSqlServer --project ...Infrastructure
      --startup-project ...API` (tool 10.0.0 vs runtime 10.0.11: benign notice).
    - **VERIFIED live**: LocalDB started, API migrated â†’ **23 tables** created,
      `__EFMigrationsHistory` stamped `InitialSqlServer`; IdentitySeeder ran
      (1 admin user + 3 sites + 4 storage tanks + 3 alert rules seeded). Login
      `superadmin@alamiot.com`/`SuperAdmin@123` â†’ 200 + JWT + full permission
      list; `/api/devices` 200 (empty), `/api/sensors/libraries` 200 (3 drivers);
      full CREATE deviceâ†’sensorâ†’telemetry tableâ†’poller insertâ†’COUNT/range query
      â†’DELETE round-trip on LocalDB OK (`telemetry_sensor_aosong_*` schema =
      bigint IDENTITY/datetime2/float/nvarchar/smallint; `GATEWAY_OFFLINE` row
      written by T-SQL; delete dropped the table). DB left CLEAN (0 devices).
      Web rebuilt/restarted :5150; login page shows `/images/logo.jpg`, routes
      302. Both builds **0 warnings / 0 errors**.
      RESTARTED: API :5080 (listener PID 7244) + Web :5150 (listener PID 10060).
26. **Alert pipeline verified + dead rules purged + metric-dropdown gated (2026-09-22)** â€”
    user reported "alerts don't fire, no notification, no mail":
    - **Pipeline PROVEN LIVE**: created a temp rule (`S-4F2F HumidityRH > 10`; current
      value 32.3) â†’ incident fired within 10s (`val=32.3, sev=Info` inserted) â†’ direct
      SMTP test with the appsettings creds (`bapz plwd ycpf pjor`) returned `MAIL SENT OK`
      and the user confirmed receipt ~1 min later. No engine/SMTP bug. Test rule + both
      incidents deleted (DB left with 0 incidents).
    - **Root cause of "nothing fires"**: DB held 3 legacy dead rules on
      `DeviceExternalId="ALL"` with metric names `Pressure` (8.5) / `FlowRate` (<10) /
      `FlowRate` (>250) â€” these NEVER match real driver columns
      (`TemperatureC`/`HumidityRH`/`InstantaneousFlowRate`/`AccumulatedTotalizer`), so
      the engine correctly ignores them. **All 3 deleted via DELETE /api/alerts/rules**
      (audit-logged). Rules now: only `S-4F2F HumidityRH > 60 Critical` (valid â€” fires
      when humidity actually crosses 60; current 32.3, correct no-op).
    - **UI gating (`Pages/Alerts.razor`)**: metric dropdown now DISABLED until a sensor
      is selected. `NewRuleTarget` defaults to `""` (placeholder "â€” Select a sensor â€”",
      ALL option moved to the bottom); `IsMetricDisabled` = empty target OR "ALL" locks
      the Metric select (shows "Select a sensor to choose parameters"); `SaveNewRuleAsync`
      guards both â€” rule cannot be saved without a concrete sensor + metric. Metric
      options still driver-aware per selected sensor (AOSONG â†’ Temp/Hum, flow drivers â†’
      Flow/Totalizer).
    - Build: Web **0 warnings / 0 errors**, restart :5150 listener PID 8328.
      RESTARTED: Web :5150 (listener PID 8328); API :5080 (PID 16416) untouched.
27. **Norvi push device registered + push pipeline re-verified (2026-09-23)** â€” the
    user's real Norvi firmware was pushing from `192.168.100.243` but NO device in DB
    had that `IpAddress`, so every push logged `Push telemetry dropped: no device
    matches source IP '192.168.100.243'` and the Norvi could never go ONLINE:
    - **Device created via API**: `Norvi ESP32 Push` (`DEV-D3CBB3D7`, `HardwareType=NorviESP32`,
      `IpAddress=192.168.100.243`, port 502, baud 9600). Note: `CreateDeviceRequest`
      takes `hardwareType`/`parity` as **strings** (`"NorviESP32"`/`"None"`), not ints.
    - **2 sensors added**: `S-F8FA` Norvi Flowmeter (Slave 1, `KAIFENG_FLOWMETER`,
      InstantaneousFlowRate + AccumulatedTotalizer) + `S-CEA3` Norvi Temp & Humidity
      (Slave 2, `AOSONG_AQ3485`, TemperatureC + HumidityRH). Create payload values with
      **ASCII-only unit strings** (`Nm3/h`, `C`) â€” non-ASCII `Â³` breaks JSON model binding
      (400 "could not be converted to System.String"). Norvi device has NO Modbus TCP
      server, so the poller skips it (HardwareType check); its online state is 100%
      push/data-driven.
    - **Push verified live (localhost test used `deviceId` fallback since source IP
      from loopback is `::1`, not `192.168.100.243`)**: `tank_id=1` â†’ accepted, DB
      `S-F8FA online=True` instantly; `tank_id=2` â†’ accepted, both sensors ONLINE with
      live values (22.5/456.7 and 47.1/30.8). Watchdog (60s) correctly flips Norvi
      OFFLINE when pushes stop (hardware currently OFF â€” DB shows offline, expected).
      Temporary test rows cleared from both telemetry tables.
    - RESTARTED: API :5080 (listener PID 14396) + Web :5150 (listener PID 15940) â€”
      both were dead when this session resumed; rebuilt + restarted.
    - **Offline-hardware log spam removed (2026-09-23)**: user has 2 Norvi + 2
      gateways; 1 Norvi + 1 gateway are routinely OFF, and every unmatched push
      printed `Push telemetry dropped: no device matches source IP ...` at **warning**
      level in the API console â€” making an EXPECTED state look like failure and
      flooding output. `PushTelemetryIngestService` now logs that drop at **Debug**
      (unknown device/sensor/driver already were Debug). The poller was already
      Debug-per-cycle (only the one-time onlineâ†’offline transition is Info). Offline
      hardware never warns anymore; a warning now genuinely means misconfiguration.
      RESTARTED: API :5080 (listener PID 11700); Web :5150 untouched.
28. **Reading "jump/miss" root cause = UI cadence, not DB (2026-09-23)** â€” user bumped
    the physical meter 46.1â†’50â†’52â†’54â†’55 and the dashboard dropped intermediate values
    ("1-2-3 readings miss kar ke phir ekdum current/last time"). Verified via TIME-GAP
    HISTOGRAM against the real per-sensor telemetry tables (`telemetry_sensor_aosong_1144a2d7`
    poll 5 s, `telemetry_sensor_kaifeng_em_5f08b5a2` poll 3 s) that the **DB has ZERO
    misses** â€” gaps are 5.3 s/3.03 s on-schedule (one 15.6 s bridge stall = tiny peak to
    peak, not a loss). The "miss" was the **UI refresh timer (4 s)** being coarser than the
    poll cadence: a 4 s frame swallows the intermediate 50/52/54 rows that the poller DID
    store horizons. Fix: `Energy.razor` (was 4000,4000) + `PlantEnergy.razor`
    (was already 2000,2000 â€” Energy was the outlier) set to **2000 ms** so the dashboard
    re-paints within ~2 s of a poll, ~every reading lands on-screen. Web build
    **0 warnings / 0 errors**. Servers left to the user (they run them: API :5080, Web :5150).
29. **Scan default = FULL bus 1..247 + live time estimate (2026-09-25)** â€” user
    asked to stop shipping the 1..10 default so a meter on a high address can never be
    missed. Changed in **all four** places: `SmartScanDtos.SmartScanRequest`
    (`EndAddress = 247`), `DevicesController.ScanBus` (`req?.EndAddress ?? 247`), Web
    client `ScanGatewayBusAsync(..., endAddress = 247)`, and `Gateway.razor`
    (`ScanEndAddress = 247`). Added `ScanEstimateSeconds` computed property, shown
    live next to the range inputs (and in the "Probingâ€¦" line) so the operator knows
    what to expect. Clamp 1..247 and registered-slave auto-skip unchanged; the range
    stays operator-editable for a fast low-range re-scan.
    - **MEASURED LIVE** (real USR-W610 @ 10.10.100.254:502, 0 registered sensors, so
      nothing was auto-skipped): `POST /api/devices/{id}/scan` body
      `{"probeTimeoutMs":200,"startAddress":1,"endAddress":247}` = **57.9 s**,
      `slavesScanned=247`, `respondingSlaves=1`, `connectivityOk=true`, found
      **slave 2 â†’ AOSONG_AQ3485 (45.8 Â°C / 30.3 %RH)**. The slave-1 EM flowmeter was
      powered off (silent) at that moment. â‡’ **~235 ms per address**, so the estimate
      formula is `addresses Ã— (probeTimeoutMs + 35)` (connect overhead is only ~35 ms,
      NOT the 110 ms first assumed). If BOTH slave 1 and 2 hold live meters the total
      stays ~58 s â€” a responder answers in ~90-180 ms, i.e. it is actually *faster*
      than a silent address that must burn the full 200 ms timeout.
    - Two red herrings investigated and dismissed (NO app bug): (a) a PowerShell
      verification script reported "login FAILED" â€” the login response property is
      `accessToken`, not `token` (`AuthController.cs:81`); (b) the scan JSON appeared
      to contain mojibake `"Â°C"` â€” that is the Windows PowerShell 5.1 console decoding
      `application/json` as Latin-1; the raw UTF-8 bytes off the wire are correct
      (`"unitPrimary":"Â°C"`), and Blazor deserializes UTF-8 properly.
    - `dotnet ef database update` re-run: **"No migrations were applied. The database is
      already up to date."** (LocalDB schema current; only the benign "EF tools 10.0.0
      older than runtime 10.0.11" notice). API + Web builds **0 warnings / 0 errors**.
    - **RUNNING NOW**: API `http://0.0.0.0:5080` (listener PID 16240), Web
      `http://localhost:5150` (listener PID 16832). Verified: login 200 (1183-char
      token), `GET /api/devices` 200 â†’ 3 devices (`USR_TST` USR-W610 @10.10.100.254
      **online, 0 sensors**; `dsadsa` Norvi @192.168.100.243 offline; `dasdsad` Norvi
      @192.168.100.56 offline), `GET /api/sensors/libraries` 200 â†’ 3 drivers
      (AOSONG_AQ3485, KAIFENG_FLOWMETER, KAIFENG_EM_FLOWMETER), all Web routes
      (`/`, `/gateway`, `/dashboard`, `/charts`, `/tanks`, `/devices`, `/reports`,
      `/alerts`) â†’ 302 `/account/login` (normal for an unauthenticated Blazor Server).
      API log has zero fail/error/warn. **Note: all 3 devices currently have 0 sensors,
      so the dashboard/Storage Tanks will be empty until sensors are mapped.**
30. **Driver #4: V880BR / LUGB VORTEX flowmeter + "same slave ID" scan warning
    (2026-09-25)** â€” user supplied a new library at
    `C:\library_own_dx\newlab\ModbusMeterLibs` (`src/Modbus.Vortex/VortexDriver.cs`,
    `VortexSettings.cs`, `settings/vortex.json`) and asked for it to be added to the
    driver library, then scanned/mapped from the UI. Second ask: when a scan skips a
    slave because that ID is already saved, show ONE clear message telling the
    operator two sensors share the ID and one must be changed.
    - **NEW DRIVER** `VORTEX_FLOWMETER` / simple `vortex` â†’
      `Domain/Drivers/VortexFlowmeterDriver.cs`, registered in `SensorDriverCatalog`
      (4th entry). Display "V880BR / LUGB Vortex Flowmeter", default slave 1, poll 3 s.
    - **Register map (per the library, bench-tested on slave 1)**: FC **0x04 input
      registers**, IEEE-754 float, HighWordFirst. 1026 qty 2 = flow **PERCENTAGE (%)**;
      1028 = sensor Hz; **1032 qty 2 = totalizer mÂ³**; 1034 overflow; 1044 magnification;
      1052 working channel (1 word); 1057 mA; 1059 pressure; 1061 temperature; 1063
      density; 1065/1067 raw pressure/temperature. Columns are the platform's standard
      `InstantaneousFlowRate` (mÂ³/h) + `AccumulatedTotalizer` (mÂ³), so charts/alerts/
      tanks/energy all work unchanged.
    - **Flow % â†’ mÂ³/h**: register 1026 is a PERCENTAGE, not mÂ³/h. Converted exactly as
      the library's cross-check does â€” `Qv = Flow% / 100 * Q20mAFullScaleFlow` with
      `Q20mAFullScaleFlow = 1000` (the meter's own "Range 100%" Basic Function item,
      default in `VortexSettings`). `FullScaleFlowM3H` is a const in the driver: change
      it if a meter's full scale differs.
    - **DRIVER CONTRACT EXTENDED for multi-window + FC04** (`ISensorDriver`): new
      `SensorReadWindow` record + `WindowTelemetry` class; new members
      `ReadWindows` (default = the single contiguous StartRegister/RegisterQuantity
      window, so the 3 existing drivers need NO change) and `ParseWindow(index, raw)`
      (default delegates to `ParseData`). `WindowTelemetry` uses **nullable**
      `PrimaryValue`/`SecondaryValue`: null = "this window doesn't feed that column"
      (so a genuine 0 reading is never confused with "not provided").
    - **WHY**: this meter's parameters are scattered and the library explicitly warns
      that ONE wide read across the undocumented gaps (e.g. 1034 â†’ 1044) corrupts the
      replies of every device on the bus. So Vortex declares TWO windows (1026 qty 2,
      1032 qty 2), each read as its own small request.
    - **Poller fixed + extended** (`ModbusPollingHostedService`): it previously ALWAYS
      called `ReadHoldingRegistersAsync` and read exactly ONE window â€” a latent FC04
      bug. Now it loops `driver.ReadWindows` (1 for existing drivers = old behaviour),
      dispatches FC03 vs FC04 per `driver.FunctionCode`, issues each window on its own
      fresh connection, merges the first non-null primary/secondary, concatenates the
      raw payloads into `RawHexBuffer`, and applies `CalibrationMultiplier` once at the
      end. Extracted `ReadWindowWithRetryAsync` (keeps the existing "retry only if the
      first attempt failed FAST" rule) + `ReadWindowAsync` (FC-aware).
    - **Scanner extended** (`ModbusScanner`): 4th signature probe (label "Vortex
      Flowmeter"); identification reads `ReadWindows[0]`, then the remaining windows
      best-effort on fresh connections so the detection card shows BOTH live values;
      `Plausible()` now takes `WindowTelemetry`. Existing behaviour unchanged.
    - **Web**: `Gateway.razor` ParamCatalog + `TankCapableDrivers` now include
      `VORTEX_FLOWMETER`; `Alerts.razor` metric dropdown offers Flow/Totalizer for it;
      `Devices.razor` tank toggle too.
    - **"Same slave ID" scan warning** (`Gateway.razor`): new `SkippedSlaveOwners`
      property joins the last scan's `SkippedSlaves` against the gateway's already-saved
      sensors (`AllChildren`) and renders a `.modal-alert warning` banner:
      "Same slave ID in use â€” two sensors are sharing one address. Slave N is already
      saved as <name> (<S-XXXX>), so that meter was skipped in this scan. Change one of
      the two IDs on the device itself, then scan again." (This complements the existing
      duplicate warning for two FOUND meters sharing an ID.)
    - Builds: API + Web **0 warnings / 0 errors** (had to stop the running processes
      first â€” MSB3026 file locks). `dotnet ef database update` â†’ "No migrations were
      applied. The database is already up to date." (no schema change; the driver
      reuses the existing two flowmeter columns).
    - **RUNNING NOW**: API :5080 (listener PID 14736) + Web :5150 (listener PID 17304).
      VERIFIED: `/api/sensors/libraries` â†’ **4 drivers**, Vortex = `fc=4 start=1026
      qty=2 m3/h m3 slave=1 poll=3`; live scan on USR_TST over slaves 1..3 (0.7 s)
      found slave 2 = AOSONG (46.3 Â°C / 30.3 %RH), `skippedSlaves: []` (nothing
      registered yet). The vortex meter is not on this bus yet, so its detection path
      is verified by catalog/compile only â€” a real bus scan is the next step.
    - **SCAN CLIENT TIMEOUT BUG (fixed 2026-09-26)** â€” clicking "Find Sensors" in the
      Gateway page appeared to fail with *"Gateway did not respond during the scan"*
      even though the gateway was ONLINE and answering perfectly. Two compounding
      causes:
      (a) `ScadaDemoTestApiClient.ScanGatewayBusAsync` used `PostAsJsonAsync` on the
      shared typed client, which is configured with `Timeout = 10s` (deliberate â€” a
      slow API must never freeze a Blazor circuit). A full `1..247` sweep genuinely
      needs **~60-63 s** (every silent address burns the whole probe timeout), so the
      request was aborted at 10 s and the bare `catch { return null; }` swallowed it.
      (b) that `null` was then folded together with a real 502 `ConnectivityOk=false`
      into one `ScanFailed` flag showing the *gateway-offline* wording, so the operator
      was told to check the power on a gateway that was up the whole time.
      **Fix**: per-request timeout override on the scan only â€”
      `request.Options.Set(new HttpRequestOptionsKey<TimeSpan>("timeout"), budget)`
      with `budget = addresses Ã— (probeTimeoutMs + 120) + 20000` (â‰ˆ99 s for 1..247 @200 ms),
      which leaves every other call on the 10 s default; plus a `ScanTimedOut` flag that
      renders a distinct amber *"The scan did not finish in time â€” the gateway itself was
      answering"* message instead of the false gateway-offline text. Also added a live
      `Scanning N s / ~M s` counter (`System.Timers.Timer` + `StopScanTimer()` in the
      `finally`) so a legitimately long sweep no longer looks like a dead button.
      Measured after the fix: full `1..247` = **63.2 s**, HTTP 200, results delivered.
31. **Vortex live on the bus + TWO CRITICAL BUGS FIXED (2026-09-25)** â€” the V880BR
    meter is now physically on the USR-W610 bus at **slave 1** and both the scan and
    the polling path are verified end-to-end on real hardware.
    - **Real bus facts** (`USR_TST` @ 10.10.100.254:502): slave 1 = V880BR vortex,
      slave 2 = AOSONG AQ3485 (~45.5 Â°C / 30.1 %RH). Slave 1's data registers
      `1026/1028/1032` (FC04) currently read all `0x0000 0x0000` â†’ **genuinely 0
      flow**, NOT a communication failure. The meter's low block is a *changing*
      status area, not flow data â€” the old temp `bus_sweep.ps1` also had an MBAP
      off-by-one, which is why slave 1 was wrongly reported "silent" earlier;
      use `bus_sweep2.ps1`.
    - **BUG 1 â€” driver was never registered with DI (poller never read it)**:
      `Program.cs` registered only 3 `ISensorDriver` singletons. `VortexFlowmeterDriver`
      was in `SensorDriverCatalog` (so `/api/sensors/libraries` showed 4) but NOT in
      DI, so the poller's `_drivers` map had 3 entries and every vortex read failed
      `UNKNOWN_DRIVER`. **The startup log line is the ground truth** â€”
      `Modbus IIoT polling worker started (drivers: ...)` must list all catalog
      drivers; if a new driver is missing there, it will never be polled.
      Fixed by adding `AddSingleton<ISensorDriver, VortexFlowmeterDriver>()`. Verify
      this log line after adding ANY driver.
    - **BUG 2 â€” scan misidentification by broad-window false positives**:
      `ModbusScanner.ProbeSlaveAsync` early-returned on the FIRST plausible decode.
      Both meters answer windows they do not own, and both decodes look plausible:
      the vortex replies to the **12-register KAIFENG** block (start 1) with its own
      dynamic status words, and the AQ3485 replies to the same block because it
      serves a contiguous register range. First-match-wins therefore flipped between
      `KAIFENG_FLOWMETER` and the truth from cycle to cycle. An intermediate
      "all-zero is ambiguous, last zero-match wins" fix was ALSO flaky (the Kaifeng
      status block is not always zero), and a naive "last match wins" fixed the
      vortex but broke the AOSONG into `KAIFENG_FLOWMETER`.
      **Final fix â€” rank all plausible matches by SPECIFICITY** (no early return):
      `(1)` narrowest identification window, `(2)` most extra windows confirmed,
      `(3)` catalog order as tie-break. The 2-register documented windows (AOSONG @0,
      VORTEX @1026) then beat the 12-register KAIFENG / 10-register EM catch-all nets
      on both meters. `RegisterQuantity` is `ushort` â€” cast to `int` in the score tuple.
    - **Inter-window gap**: RS-485 is half-duplex, so back-to-back register-block
      requests collide and replies are lost. `InterWindowDelayMs = 60` is now applied
      between extra windows in BOTH `ModbusPollingHostedService` and
      `ModbusScanner` (the library's own `InterReadDelayMs` is 50). The scanner's
      delay must fire before the FIRST extra window (`w > 0`), not `w > 1`.
      Every scanner signature probe also opens its own fresh TCP connection.
    - **VERIFIED LIVE**: 3 consecutive scans over 1..3 each returned
      `slave 1 â†’ VORTEX_FLOWMETER 0 mÂ³/h / 0 mÂ³` and
      `slave 2 â†’ AOSONG_AQ3485 45.6 Â°C / 30.1 %RH`, `respondingSlaves=2`,
      `unknownResponders=0`, `skippedSlaves: []`. Full `1..247` scan = **57.4 s**
      (no regression vs the 57.8 s baseline; silent addresses still cost one read).
      A temporary sensor (`S-6902`, `telemetry_sensor_vortex_6b110d39`, slave 1,
      poll 3 s) streamed real telemetry at ~3.5 s cadence, `ConnectionStatus=1`,
      no error codes, sensor `IsOnline=true` â€” then was **deleted, DB left CLEAN
      (0 sensors)**. `/api/sensors/libraries` â†’ 4 drivers. All Web routes 302.
      API log has zero warn/error/fail. Builds **0 warnings / 0 errors**.
    - **RUNNING NOW**: API :5080 (listener PID 18108) + Web :5150 (listener PID 1900).
    - **Debugging tip that actually worked**: per-sensor poll failures are
      `LogDebug`, so nothing appears in a normal API log. Restart the API with
      `set Logging__LogLevel__Default=Debug` (appsettings pins the EF category to
      Information, so there is no SQL flood) to surface
      `Sensor '<name>' on '<device>' poll failed: ...`. A device that produces NO
      telemetry rows AND NO offline rows usually means the sensor never reached the
      read path at all â€” check the driver registration log line first.
32. **Driver #5: Selec RI-F200-C power/energy meter + ZERO-FILL BUS FIX (2026-09-26)**
    â€” user supplied a new library at
    `C:\final_hx_librery_test\hienergy _test` (a `PlantPulse` project; the meter
    profile lives in `ModbusGatewayOptions.cs` / `ModbusGatewayPollingHostedService.cs`
    and `appsettings.json` gives slave id **5**) and asked for it to be added to the
    driver library, then scanned/mapped from the UI.
    - **NEW DRIVER** `SELEC_POWER_METER` / simple `selec_power` â†’
      `Domain/Drivers/SelecPowerMeterDriver.cs`, registered in `SensorDriverCatalog`
      (5th entry) + DI + `Program.cs` and in the `Gateway.razor` / `Alerts.razor`
      parameter lists. Display "Selec RI-F200-C 3-Phase Power/Energy Meter",
      default slave 5, poll 5 s.
    - **Register map (per the reference library)**: FC **0x04 input registers**,
      IEEE-754 float, **LOW-WORD-FIRST** ("FLOAT REVERSE WORD" â€” first register on
      the wire is the LOW word; the exact inverse of the Kaifeng/vortex high-word-
      first helper). Core block @ 0 qty 74, THD @ 122 qty 20, demand @ 692 qty 28;
      **NO +1 PDU offset**. Total active power = float 21 â†’ **regs 42,43 (kW)**;
      active energy totaliser = float 29 â†’ **regs 58,59 (kWh)**. Two isolated
      2-register windows (the library itself reads these meters in 4-register
      chunks, because a wide 74-register read over a serial bridge comes back
      short/garbled). Columns are the platform's standard
      `InstantaneousFlowRate`/`AccumulatedTotalizer`, so charts/alerts/reports/dashboard
      work unchanged; the UNITS are kW / kWh.
    - **Driver contract extended â€” `ISensorDriver.CorroborationWindow`**
      (a `SensorReadWindow?`, default `null`). It is used ONLY by the scanner and is
      deliberately NOT part of `ReadWindows`, so it never costs the polling worker an
      extra request. Selec declares `start 64, qty 10` â€” inside its own documented
      74-register core and claimed by NO other driver in the catalog.
    - **THE BUS PROBLEM THIS SOLVES (important)**: the RI-F200-C answers **EVERY**
      function-04 address with zeros instead of raising `0x02` illegal-data-address
      (verified: `@90`, `@122`, `@400`, `@684`, `@692`, `@2000` all reply with byte
      counts and zeros), and its low core block decodes to plausible floats whose
      register 0 is *sticky-but-changing* (`0x008F` â†’ `0x00EC` across reads, reg 1
      `0x0068` stable). Consequence: on this bus the scan confidently reported the
      energy meter as **`VORTEX_FLOWMETER 0 mÂ³/h / 0 mÂ³`** â€” a genuine
      misidentification, because the real V880BR also serves 1026/1028/1032 and also
      reads all zeros there.
    - **Two rejected heuristics â€” do not reintroduce them**:
      (a) counting only NON-ZERO replies as "confirmed windows" BREAKS the verified
      vortex path, because the real V880BR genuinely reads `0/0` at 1026/1028/1032;
      (b) a two-consecutive-read STABILITY check cannot separate the two either,
      because the meter's dynamic register is sticky and consecutive reads agree.
      Both were tried, measured, and reverted.
    - **The fix that works** â€” `ModbusScanner` now (1) REQUIRES the corroboration
      window to be *served* (a reply, not an exception) for a signature to count at
      all, and (2) ranks `Unproven = corroborated ? 0 : 1` immediately after `Width`,
      i.e. ahead of `Extra`/`Order`. Counting the corroboration block inside
      `confirmedWindows` alone was NOT enough: Selec (42, 58, +corroboration) and
      Vortex (1026, 1028, 1032) both scored 3 and the catalog-order tie-break handed
      it to Vortex. Proof must outrank a mere window count.
    - **VERIFIED LIVE (2026-09-26, all 3 meters on the bus at once)**: once the
      USR-W610 was powered back on, `1..8` returned **3 of 3 correct** matches on
      **3 consecutive runs** (~9.7 s each, `respondingSlaves 3`, `connectivityOk true`):
        - slave **1 â†’ `VORTEX_FLOWMETER`** â€” V880BR, 0 mÂ³/h / 0 mÂ³
        - slave **2 â†’ `AOSONG_AQ3485`** â€” 56.5 Â°C / 30.3 %RH (live, real bus data)
        - slave **5 â†’ `SELEC_POWER_METER`** â€” Selec RI-F200-C, 0 kW / 0 kWh
      This is the decisive test: the vortex-vs-Selec arbitration is now confirmed
      **with both devices present**, which is exactly the combination that used to
      flip between `VORTEX_FLOWMETER` and the truth. Full `1..247` scan = **59.4 s**.
      `/api/sensors/libraries` â†’ 5 drivers; startup line lists all 5
      (`... VORTEX_FLOWMETER, SELEC_POWER_METER`). API + Web build **0 warnings /
      0 errors**. DB left CLEAN â€” **0 sensors**, nothing was auto-created by the scan
      (device `USR_TST` / `DEV-D95D6B02` shows `online=True` with a fresh
      `LastSeenAt`, so the poller is watching the bus).
    - **STILL OPEN â€” the meter's own inputs**: the energy meter reads **0 kW / 0 kWh**
      and its whole documented block is zero, while the reference project saw
      frequency â‰ˆ 49.8 Hz on a properly powered unit. Its voltage/CT inputs are almost
      certainly still not connected, so no non-zero energy value has ever been seen and
      the low-word-first float decode remains unproven on real data. Also the two Norvi
      devices (`dsadsa` @192.168.100.243, `dasdsad` @192.168.100.56) are still OFFLINE â€”
      expected, they are push-driven and not pushing.
33. **AUTO bus scan â€” all connected meters found, ZERO operator knobs (2026-09-26)** â€”
    user directive: *"jitni bhi connect ho sub show kar"* â€” every connected meter must
    appear, and the operator must not have to tune anything. This replaces the
    knob-heavy range/timeout/passes form.
    - **ROOT CAUSE of "found nothing" (proved, not guessed)**: the scan is ONE HTTP call
      that legitimately needs ~60 s (every silent address burns the whole 200 ms probe
      timeout), but Web's shared `HttpClient` caps at **10 s**. The previously added
      per-request override was **measured to be ignored** â€” a repro through a 10 s client
      died at exactly `10.0 s` with `TaskCanceledException` both with and without
      `request.Options.Set(new HttpRequestOptionsKey<TimeSpan>("timeout"), ...)`. The
      client's own `Timeout` always wins. Silent `catch { return null; }` then surfaced
      as a false "gateway did not respond".
    - **Fix = a dedicated client**, not an override. `Web/Program.cs` registers a named
      client `scada-bus-scan` (5 min cap, same bearer-forwarding handler);
      `ScadaDemoTestApiClient` now takes `IHttpClientFactory` and uses that client for
      the scan only, sizing `Timeout = addresses Ã— (probeTimeout + 150) Ã— passes + 30 s`.
      Every other call stays on the 10 s safety cap.
    - **AUTO tiering in `ModbusScanner`** (`Passes <= 0` â‡’ auto, the default everywhere):
      - `AutoPasses = 3` â€” a serial bridge drops real meters mid-sweep, so ONE pass is
        not trustworthy (measured: 3/3 meters on one pass, 1/3 on the next).
      - `QuickScanEndAddress = 16` â€” tier 1 sweeps `1..16 Ã— 3 passes` (48 probes,
        **~23-27 s measured**) which is where meters on a real plant bus actually live.
      - tier 2 = the **full `1..247 Ã— 3`**, and it runs **only when tier 1 found
        NOTHING** (a partial result already proves the bus is alive, so widening would
        just burn minutes).
      - Results are merged **by slave address** across passes/passes-and-tiers, so a
        meter is never listed twice and never drops out between passes. The merge keeps
        the best (identified) match but always takes the **freshest live values**.
    - **UI simplified to one button** (`Gateway.razor`): the "Address range 1..to",
      "Probe timeout (ms)" and "Passes" inputs are **gone**. `ProbeTimeoutMs` is a
      `const 200` (9600-baud meters measured 90-107 ms round-trip, so the old 20 ms probe
      always missed them) and the estimate text now only explains the automatic sweep.
      `Scanning N s / ~M s` timer + the distinct amber "did not finish in time" message
      are kept.
    - **VERIFIED LIVE on the real USR-W610** (`USR_TST` / `DEV-D95D6B02`,
      10.10.100.254:502, 0 registered sensors): **3 consecutive AUTO runs, 3/3 meters
      every time, `unknownResponders = 0`, `connectivityOk = true`, 23.1 / 23.3 / 27.1 s**:
        - slave **1 â†’ `VORTEX_FLOWMETER`** â€” 0 mÂ³/h / 0 mÂ³
        - slave **2 â†’ `AOSONG_AQ3485`** â€” 54.9 Â°C / 30.3 %RH
        - slave **5 â†’ `SELEC_POWER_METER`** â€” 0 kW / 0 kWh
      API + Web build **0 warnings / 0 errors**; `/_framework/blazor.server.js` 200;
      `/api/sensors/libraries` 200 (5 drivers); logs clean.
    - **STARTING THE SERVERS CORRECTLY (a real trap)** â€” running the built `.exe` from
      an arbitrary working directory **silently breaks it**: ASP.NET Core takes the CWD
      as content root, so `appsettings.json` is not found â†’ **no connection string** â†’
      every request 500s with `InvalidOperationException: The ConnectionString property
      has not been initialized`. And without `--urls` the API listens on **5000**, not
      5080. Correct launch (what the earlier `dotnet run` did via `launchSettings.json`):
      ```
      Start-Process -FilePath "$apiDir\scada_demo_test.API.exe" `
        -ArgumentList "--urls","http://0.0.0.0:5080" -WorkingDirectory $apiDir
      Start-Process -FilePath "$webDir\scada_demo_test.Web.exe" `
        -ArgumentList "--urls","http://localhost:5150" -WorkingDirectory $webDir
      ```
    - **RUNNING NOW**: API `http://0.0.0.0:5080` (listener PID 13420), Web
      `http://localhost:5150` (listener PID 15596), both started with
      `ASPNETCORE_ENVIRONMENT=Development` and a correct `-WorkingDirectory`.
      DB still has **0 sensors** on this gateway (scans never auto-create; the operator
      maps a found meter with "Add Sensor to Application").
    - **PowerShell gotcha that wasted a cycle**: `Invoke-RestMethod` returns an ARRAY and
      PS 5.1 member enumeration makes `$_.ipAddress -eq "<ip>"` match EVERY element, so
      `$dev` became all 3 devices and the scan URL 404'd. Use an explicit
      `foreach ($x in @($all)) { if ($x.ipAddress -eq "...") { $dev = $x } }`.
34. **COMPLETE scan (the "50 meters" bug) + offline pre-check + scan spinner (2026-09-26)** â€”
    user asked: *"agar 50 meter connect kar ke find karu to find hongi?"* and *"offline
    device pe scan karo to scanning na lage, device offline msg do"*.
    - **THE DENSE-BUS BUG (real, would have silently dropped meters)**: the tiering
      introduced above stopped after tier 1 as soon as ANYTHING answered
      (`if (tier + 1 < tiers.Count && bySlave.Count > 0) break;`). On a bus with 50
      meters over addresses 1..50, tier 1 (1..16) found 16, the scan stopped, and
      addresses 17..50 â€” **34 meters â€” were never probed at all**. "Find every connected
      meter" can never stop early. Tier 2 was also `1..247`, wasting time re-probing
      1..16.
      - **Fix**: tier 2 is now `(QuickScanEndAddress + 1)..MaxAddress` = **17..247**, so
        the tiers are disjoint and together cover **every** address, and **both tiers
        always run** (the early `break` is deleted). `Passes = 1` keeps a quick-only
        path (tier 1) for operators who accept the trade-off.
      - **VERIFIED LIVE**: `passes=0` (complete) probed **741 addresses = 247 Ã— 3 exactly,
        225.6 s**, and found **3/3** (`slave 1 VORTEX`, `slave 2 AOSONG 70.1 Â°C / 30.8 %RH`,
        `slave 5 SELEC`) with `unknownResponders = 0`. The exact 741 count is the proof
        that no address is skipped, which is what makes "however many are connected,
        all are found" true.
    - **Offline pre-check (new endpoint)**: `GET /api/devices/{id:reachability}`
      (`DevicesController.Reachability`, `Action:{AppPermissions.DevicesView}` â€” the
      policy value is the dotted string `"Devices.View"`, NOT `"DevicesView"`, or startup
      500s with *The AuthorizationPolicy named 'Action:DevicesView' was not found*).
      It is ONE `TcpClient.ConnectAsync` with a 1.5 s `CancellationTokenSource` cap and
      returns `{ reachable, online, latencyMs, message }`. It intentionally ignores the
      worker-maintained `IsOnline` (which can be a watchdog interval stale) and probes
      live. `Gateway.razor` calls it **before** setting `Scanning = true`, so an offline
      gateway shows a red "Device offline" line instantly instead of a spinner that can
      only end in failure. MEASURED: online gateway `reachable=true` in **3 ms**; the
      offline Norvi `reachable=false` in **1504 ms** with *"Gateway 192.168.100.243:502
      is offline (no response within 1.5 s)."* â€” previously this cost a full sweep.
    - **Spinner**: `.sc-scan-line` / `.sc-spinner` (15 px, `sc-spin` 0.7 s linear) in
      `site.css`, with a `.sm` 12 px variant rendered inside the Find button via
      `(MarkupString)` (`Html.Raw` fails: no `@using Microsoft.AspNetCore.Components.Web`
      on the page â†’ CS0103). Includes a `prefers-reduced-motion` fallback.
    - **Two buttons, no knobs**: **ðŸ” Find Sensors** = COMPLETE (`passes=0`, whole bus,
      ~3.8 min measured 225 s) â€” the default, guaranteed to find every connected meter;
      **âš¡ Quick 1-16** (`passes=1`, ~25 s) â€” explicitly labelled and tool-tipped as a
      fast look that *can* miss a meter above address 16. Both buttons sit in the
      collapsed card and the Quick one only appears once the gateway is expanded.
    - **Building the Web project in a `if/else` expression** returns `object` and works,
      but `Html.Raw` is unavailable â€” cast to `(MarkupString)`.
    - **The gateway went offline after the 741-probe complete scan** (`ping
      10.10.100.254` = False, so it is off the network, not just the port). Not an app
      bug: the API correctly answered `502`/`reachable=false`. Worth noting that a very
      long sustained sweep can be enough to knock a serial bridge over, so if a
      3.8-minute complete scan is followed by an offline gateway, power-cycle it before
      blaming the software.
    - **Servers MUST be started detached** (WMI), otherwise the tool's own process-tree
      cleanup kills them seconds after they boot and every later call gets
      "Unable to connect to the remote server":
      ```
      Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
        CommandLine = "cmd.exe /c cd /d `"$apiDir`" && set ASPNETCORE_ENVIRONMENT=Development && scada_demo_test.API.exe --urls http://0.0.0.0:5080" }
      ```
    - Builds: API + Web **0 warnings / 0 errors**. Reachability + complete-scan paths
      verified live. `passes=1` (Quick) is compile-verified only â€” the gateway went
      offline before it could be measured, so re-verify when the gateway is back.
    - **RUNNING NOW** (WMI-detached): API `http://0.0.0.0:5080` (listener PID 13052), Web
      `http://localhost:5150` (listener PID 12160).
35. **QUICK = 1..20, per-address isolation, wedged-bridge finding (2026-09-26)** â€”
    user asked *"quick mode pe 1 se 20 slave id scan ho aur find sensor pe 1 se 247"*.
    - **Ranges now fixed in code, no operator input**:
      `QuickScanEndAddress = 20`. `passes = 1` â†’ QUICK = **slave ids 1..20, 3 passes**
      (~15 s). `passes <= 0` â†’ "Find Sensors" = **1..247, 3 passes**, split into the
      disjoint tiers `1..20` + `21..247`.
    - **BUG FIXED â€” the Quick button scanned the WHOLE bus.** `quickOnly` only trimmed the
      AUTO tier list, so a non-AUTO request (`passes = 1`) fell through to the caller's
      explicit range and swept `1..247` (measured **244 probes / 67 s** instead of 60).
      The tiers are now chosen by a single switch on `(autoMode, quickMode)`.
    - **BUG FIXED â€” one bad address aborted the whole sweep (silent meter loss).** A single
      socket hiccup inside `ProbeSlaveAsync` propagated to the outer `catch`, which set
      `GATEWAY_UNREACHABLE` and **stopped the scan**. Measured: a `1..8` scan probed only
      **5** addresses and reported **0 meters** on a perfectly healthy bus. Each address is
      now isolated in its own try/catch; the sweep skips the failed address and continues.
      The gateway is only declared unreachable after
      `UnreachableFailureStreak = 12` consecutive failures (one failed probe is normal on a
      a busy serial bus), which then does `goto done` and returns the partial result.
    - **IMPORTANT HARDWARE FINDING â€” the USR-W610 can wedge after a long sweep.** After
      the 741-probe complete scan the gateway first went off-network entirely
      (`ping = False`), then on a later attempt it came back as: `ping = True`, TCP `:502`
      **accepts connections**, reachability endpoint says `reachable=true` in 2-3 ms, but
      a raw Modbus FC03 read never returns â€” `Unable to read data from the transport
      connection: ... the connected host has failed to respond`, and the scan reports
      `probes=12 responding=0 connectivityOk=false err=GATEWAY_UNREACHABLE`. So
      **`reachable=true` only proves the TCP listener is alive, not that the serial bridge
      serves Modbus.** A power-cycle of the gateway is the fix; do not debug the scanner
      for it. Expect this after a multi-minute full-bus sweep.
    - Builds: API + Web **0 warnings / 0 errors**. Range change + Quick label verified by
      build; Quick/full live runs pending a gateway power-cycle (the bridge was wedged).
    - **RUNNING NOW** (WMI-detached): API `http://0.0.0.0:5080` (listener PID 8716), Web
      `http://localhost:5150` (listener PID 3632).
36. **Added sensors are ALWAYS a visible child of their device (2026-09-26)** â€” user:
    *"jb ek device se koi bhi sensor add to application ho gaya, to wo us device ka child
    ban kar nazar ata rahe"*. The **Gateway** page already rendered children always (via
    `AllChildren`), but the **Devices / Norvi** page (`/devices`) still loaded children
    only for the ONE expanded device (`ExpandedDeviceId`) and rendered them inside
    `@if (expanded)` â€” so every other device's sensors were invisible until you clicked
    the row open, and a just-added sensor looked like it had vanished.
    - **`Devices.razor`**: new `AllDeviceSensors` dict (deviceId â†’ children) is rebuilt
      from ONE `GetSensorsAsync()` (no-arg) call inside `LoadDevicesAsync`, which the 1 s
      refresh timer already runs, so children are always fresh; it also back-fills
      `DeviceSensorCache`. `ChildrenOf(Guid)` helper returns `null` until loaded (so a
      still-loading device does not flash a wrong "no sensors" state).
    - **Render moved OUT of `@if (expanded)`**: the "Attached Sensor Children (n)" tree now
      renders for EVERY device unconditionally, with the same `sensor-row-highlight`,
      live values, ONLINE/OFFLINE pill and Delete Sensor button as before. Only the
      `+ Add Sensor` button stayed behind the expand click. Razor gotcha hit: a
      `@{ var childList = ...; }` code block is **not allowed inside a `@foreach`** body
      (RZ1010) â€” use the pattern matcher `@if (ChildrenOf(d.Id) is { Count: > 0 } childList)`
      instead.
    - Verified data-wise: `GET /api/sensors` returns 3 sensors, all with the correct
      `deviceId` = `USR_TST` (`7a28b7b8-â€¦`), `sensorCount=3` on the device DTO â€” so the
      grouping key matched; the missing children were purely the page's expand gate.
    - Build: Web **0 warnings / 0 errors**; `/_framework/blazor.server.js` 200.
    - **RUNNING NOW** (WMI-detached): API `http://0.0.0.0:5080` (listener PID 8716), Web
      `http://localhost:5150` (listener PID 3632).
37. **METER IDENTIFICATION CORRECTNESS + AMBIGUITY/CONFIRMATION (2026-09-28)** â€” the
    scanner was misidentifying BOTH real meters on the USR-W610 bus. Fixed, and the
    remaining core directive (never guess when the bus cannot decide) is now
    implemented and proven on a purpose-built mock.
    - **Ground truth (verified on `USR Gateway` @ 10.10.100.254:502)**: slave `2` =
      `AOSONG_AQ3485` (live Â°C/%RH); slave `5` = `SELEC_POWER_METER` (live ~234 V
      three-phase voltage, 0 kW/0 kWh, ~50.1 Hz, THD ~3.7 %, PF 1). Slaves
      `1,3,4,6,7,8` are EMPTY and the gateway replays slave 2's cached frame â€”
      including unit ID `2` â€” for them. **No Vortex, EM or thermal-mass slave ID has
      ever been found on this bus**; do not guess one from numeric values.
    - **The empty-address trap (unit ID validation, keep this)**: a reply whose MBAP
      unit ID is not the address probed proves the frame belongs to another device.
      `ModbusTcpMaster` now rejects it (`ModbusException.IsProtocolError`). The
      scanner maps that to `WindowInvalid` and CONTINUES. An earlier attempt treated
      a unit-ID mismatch as `GATEWAY_DESYNCED` and aborted the whole sweep â€” that was
      **wrong** and was removed: a mismatch is the normal signature of an empty
      address on this gateway, not a protocol fault.
    - **FC43 (MEI 0x0E) and FC11 tested â€” both UNUSABLE for identification here.**
      Slave 5 returns a proper `EXC 0x01`; every other address replays the cached
      AQ3485 frame, so there is no clean unit-ID/framing discriminator to exploit.
      Don't retry this route.
    - **THE SCORING FIX (the actual bug)**: candidates were ranked with live-evidence
      BEFORE specificity, so a humidity sensor answering the 12-register thermal-mass
      block (stray non-zero words) beat the real 2-register AQ3485 signature on
      slave 2 â€” measured `(unproven 1, live -2, width 12, â€¦)` vs `(1, -1, width 2)`.
      Correct order is now, lowest-wins:
      `(Width, Unproven, Live, Extra, Order)` in `CandidateRank`
      (`ModbusScanner.cs`, a nested `readonly record struct` so the probe can hand its
      quality back out):
      1. `Width` â€” narrowest identification window = most specific signature. A
         documented 2-register window is categorically different from a 12-register
         catch-all block.
      2. `Unproven` â€” a driver whose OWN `CorroborationWindow` was served **with
         data** outranks one that could not prove itself. This is what takes slave 5
         from the AQ3485: the RI-F200-C answers the AQ3485's 2-register block with
         its CT primary rating `(1000, 0)`, which decodes to a plausible
         `100 %RH / 0.0 Â°C`, and only the power meter can serve live three-phase
         voltage from `@0/4`.
      3. `Live` â€” most of the driver's own blocks carrying real content.
      4. `Extra` â€” most extra blocks served WITH DATA. Zeros are deliberately NOT
         counted; counting them is exactly what let a humidity sensor win as a gas
         flowmeter.
      5. `Order` â€” catalog order, the final deterministic tie-break.
    - **Degeneracy guard**: a driver is rejected when all of its own decoded values
      are exactly zero **and** it could not be corroborated. Without the
      `!corroborated` exemption this kills the real power meter, whose kW/kWh
      legitimately read `0` and which identifies itself ONLY through its voltage
      block.
    - **Corroboration read is retried once** (`InterWindowDelayMs` between attempts).
      A serial bridge under sweep load intermittently drops a real meter mid-pass, and
      losing that single read collapsed the whole RI-F200-C match.
    - **MERGE ACROSS PASSES IS NOW RANK-AWARE (this was the flip)** â€” `bySlave` kept
      whichever pass answered FIRST, so one unlucky pass permanently pinned a wrong
      driver to an address and no later pass or re-read could undo it (this is how
      slave 5 kept coming back as `AOSONG_AQ3485` with `passes=3`). It now keeps the
      lower `CandidateRank` and only refreshes live values.
    - **VERIFIED LIVE, 4 consecutive `1..20 Ã—3 passes` runs, all identical**:
      `S2 -> AOSONG_AQ3485 29.3 Â°C / ~50 %RH`, `S5 -> SELEC_POWER_METER`, in
      ~20-25 s per run, `unknownResponders=0`. `ambiguousSlaves=0` â€” correct, the real
      bus IS decidable.
    - **AMBIGUITY + OPERATOR CONFIRMATION (new, verified both directions)**: a slave
      is flagged `IsAmbiguous` **iff the top two candidates tie on BOTH `Width` and
      `Unproven`** â€” i.e. the winner was decided only by how much data it scraped,
      which is precisely the tie-break that flips between passes. A different width or
      a different proof verdict is a hard discriminator and is NOT ambiguous.
      - `ScannedMeterDto` gained `IsAmbiguous` + `Candidates`
        (`ScannedCandidateDto`: driver key/display, window, live values,
        `ProofServed`); `SmartScanResultDto` gained `AmbiguousSlaves`. Web client DTO
        mirrors both.
      - `Gateway.razor` renders an amber per-row "Ambiguous â€” N meter types match this
        address. Confirm which one is physically connected" banner with one button per
        driver, the best guess labelled "(best guess)", plus a summary banner when
        `AmbiguousSlaves > 0`. The choice is stored in `_chosenDriver` (slave â†’ key) and
        `Resolved(m)` rebuilds the meter from the CONFIRMED driver, deliberately
        clearing `AvailableParameters`/`LiveValues` so the parameter list comes from the
        new driver and never from the guess being replaced. `ParamsFor()` also keys off
        the confirmed driver. `_chosenDriver` is cleared on a new scan and on ClearScan,
        and changing the choice drops any already-picked meter for that slave.
      - **The "lock" needs no new schema**: once the operator saves, the `Sensor` row
        holds `SensorTypeKey` + `SlaveAddress`, and the polling worker reads that row
        directly â€” it never re-identifies. The confirmation is only needed at scan
        time, and the sensor row IS the persisted lock.
      - **PROVEN on a purpose-built mock Modbus gateway**
        (`%TEMP%\opencode\ambmock`, slave 1 on 127.0.0.1:12599, sparse register map,
        exception `0x02` for unserved ranges and `0x0B` for other unit IDs). It is
        built to force exactly the fragile tie: AOSONG FC03 `@0/2` = 55.5 %RH/25.5 Â°C
        and VORTEX FC04 `@1026/1028/1032` all live, with the Vortex `CorroborationWindow`
        `@1067/2` served as ZEROS. Result: `ambiguousSlaves=1`, `S1 driver=
        VORTEX_FLOWMETER (250 mÂ³/h, 123.46 mÂ³) IsAmbiguous=True`, alternative
        `AOSONG_AQ3485 25.5/55.5 proofServed=False`. Flipping only the proof block to
        live data made the SAME scan return `ambiguousSlaves=0`, Vortex, no ambiguity â€”
        so both the positive and negative cases are verified, not just the happy path.
        Temp mock device created for the test and **deleted**; `0` orphan tables.
    - **Selec slave 5 mapped and streaming**: sensor `S-F0C1`, driver
      `SELEC_POWER_METER`, slave 5, poll 5 s, **all 61 parameters** selected, table
      `telemetry_sensor_selec_power_7e6cd976`. Live rows carry
      `VoltageV1Nâ‰ˆ234.5 V`, `Frequencyâ‰ˆ50.1 Hz`, `ThdV1Nâ‰ˆ3.8 %`, `PF1=1`,
      `CurrentI1=0 A` (CT not connected â€” the meter's own input, not an app fault),
      `ConnectionStatus=1`, no error codes, `IsOnline=true`, cadence ~10-13 s under the
      load of a concurrent full scan. This **proves the Selec low-word-first float
      decode on real data**, which was previously unverified.
      NOTE: the Create DTO fields are `SensorName`/`DeviceId`/`SensorTypeKey`/
      `SlaveAddress`/`PollIntervalSeconds`/`CalibrationMultiplier`/`SelectedParameters`/
      `SensorConfig` (PascalCase) â€” lower-case `name:` returns
      `400 The SensorName field is required`. Non-ASCII unit strings (e.g. `Â°C`) also
      break model binding; send `C`/`m3`.
    - Build: API + Web **0 warnings / 0 errors**; API/Web logs 0 warn/error/exception;
      `/_framework/blazor.server.js` 200. Debug-only candidate logging was left in
      (`ILogger<ModbusScanner>`, all at `LogDebug`) because per-address scoring is
      otherwise invisible â€” surface it with
      `set Logging__LogLevel__Default=Debug`.
    - **RUNNING NOW** (WMI-detached): API `http://0.0.0.0:5080`, Web
      `http://localhost:5150`, both `ASPNETCORE_ENVIRONMENT=Development`.
      DB: 3 devices (`USR Gateway` online + 1 sensor, `Norvi 56` / `Norvi 243`
      offline and push-idle as expected), 1 sensor (`S-F0C1`).


38. **COMMUNICATION HEALTH / QUALITY INDICATORS (2026-09-28)** - user asked for
    a per-device and per-sensor health verdict on every Modbus request, live
    real-time push, and interactive "What's This?" tooltips on the devices/sensors
    list. **COMPLETE END-TO-END, VERIFIED LIVE (backend + UI).**
    - **Quality ladder** (`Domain/Enums/QualityCode.cs` + the rule engine in
      `Domain/Health/HealthThresholds.cs`, kept as ONE auditable pure function so
      the in-memory hot path and any re-evaluation apply byte-identical logic):
      `GOOD` = correct data inside a **500 ms** latency budget (measured: the real
      AQ3485 round trip is 90-107 ms, so 500 ms leaves real headroom);
      `UNCERTAIN` = latency > 500 ms OR 1-2 drops in the last 5 polls;
      `BAD` = **3 consecutive failures**, or **immediately** on a Modbus
      EXCEPTION frame (the meter is powered and talking but refusing the request â€”
      a register-map/function-code fault, so it must not wait for a streak);
      `STALE` = nothing for 3x the target's own poll interval (distinct from BAD:
      "no fresh data", not "a fault we can see"); `UNKNOWN` = never polled.
      Precedence is `STALE freshness` -> `BAD` -> `UNCERTAIN` -> `GOOD`, so BAD
      always outranks a slow-but-arriving reading.
    - **The rolling window** (`Infrastructure/Health/CommunicationHealthTracker.cs`,
      singleton): a **fixed 100-slot bool ring per target**, so memory is bounded
      and success rate / drop count are true *sliding* windows, not lifetime
      averages. Per-target `lock`, so the poller never takes a global lock across
      gateways. `Stale` needs no background loop at all â€” it is re-evaluated on
      every read from `LastPollAtUtc`, so a target that simply stopped being polled
      degrades to STALE on its own. (`RefreshStaleness` was removed as dead code.)
    - **The poller hook** (`ModbusPollingHostedService`): `ReadWindowWithRetryAsync`
      now returns `(Payload, WindowOutcome)` instead of a bare `byte[]`, so every
      Modbus request yields a classified outcome. `WindowOutcome.FromException`
      maps `ModbusException{ExceptionCode != null}` -> `ModbusException`,
      other `ModbusException` -> `ProtocolError`, `TimeoutException` ->
      `Timeout`, `SocketException`/`IOException` -> `ConnectionRefused`. The
      reconnect retry is NOT a second poll, so the latency is the wall-clock of the
      whole window and a window that succeeded on retry is one success. A sensor
      with N selected windows is **ONE poll**, not N. A poll that threw is recorded
      as one failure for BOTH the sensor and its gateway, so a single sick slave
      shows UNCERTAIN on the device badge as well. A 0-sensor gateway's reachability
      probe IS its health sample, so a healthy gateway reads GOOD before any meter
      is attached. A gateway whose sensors are all not-due is still LIVE, so it gets
      a GOOD verdict rather than falsely reading STALE.
    - **Hot path is untouched**: `Record()` is pure memory. Persistence is a
      separate `HealthSnapshotPersistenceService` (5 s `PeriodicTimer`) that upserts
      snapshots inside `EfResilience`. A slow/failing DB write delays nothing and
      the badge is still served from memory. It resolves the SCOPED
      `IHealthRepository` through its own `IServiceScopeFactory` scope â€” injecting
      it directly is a startup-time DI validation failure
      ("Cannot consume scoped service from singleton IHostedService").
    - **Realtime**: `IHealthBroadcaster` + `SignalRHealthBroadcaster` on the SAME
      hub as telemetry, so one connection carries both; `ReceiveHealth` /
      `ReceiveHealthAll`. Kept a separate interface from `ITelemetryBroadcaster`
      because the two transports can change independently.
    - **API**: `GET /api/health` -> `HealthOverviewDto` (devices + sensors +
      `HealthSummaryDto` roll-up + the threshold constants so the UI can label its
      own legend). Falls back to the persisted snapshot and seeds the tracker when
      the in-memory state is empty, so a dashboard opened right after a restart
      shows real numbers instead of "never polled".
    - **DB**: `CommunicationHealth` entity (snapshot per `(Kind, TargetId)`, unique
      index) + migration **`20260928110016_AddCommunicationHealth`**, APPLIED to
      LocalDB. Deliberately a snapshot table, NOT a second telemetry store â€” the
      per-sensor telemetry tables already hold every reading.
    - **EF method-name trap (cost real time â€” do not repeat)**: the override is
      **`OnModelCreating(ModelBuilder)`**, NOT `OnModelBuilder`/`OnCreating`. My
      first `Write` pass silently renamed it and produced
      `CS0115 'no suitable method found to override'` for ~10 build cycles. Also
      note this project's DbSet alias is `Dbset<>` (lowercase s), the context class
      is `MyDbContextDxy`, and the parameter must be lowercase `modelBuilder`.
      **When a camelCase token must be preserved exactly, verify the file's raw
      bytes (Format-Hex) before building** â€” the editor normalises tokens silently.
    - **VERIFIED LIVE** on the real `USR Gateway` @ 10.10.100.254:502 â€”
      `GET /api/health` -> **HTTP 200**, `USR Gateway` (`DEV-A355817F`) = **GOOD**,
      latency **2.2 ms**, success **100%**, 20 samples, last poll current; summary
      `1 good / 0 uncertain / 0 stale / 0 bad`. The classifier was proven against
      all 4 states + recovery on a throwaway harness (10 fast -> Good; 900 ms ->
      Uncertain; 3 timeouts -> Bad; 1 drop + recovery -> Uncertain with
      `DropsInLastWindow=1`; Modbus EXCEPTION -> Bad on the FIRST strike; 5 clean ->
      Good; seeded 10 min old -> Stale; never polled -> null). Note a recovered
      target legitimately keeps a sub-100% success rate for a while, because the
      100-slot ring still holds the old failures â€” that is the sliding window
      working, not a bug.
    - **Policy gotcha reconfirmed**: `[Authorize(Policy = AppPermissions.DevicesView)]`
      500s with *"The AuthorizationPolicy named 'Devices.View' was not found"*; the
      policy NAME is **`"Action:" + AppPermissions.DevicesView`**.
    - **UI (DONE)**: two reusable components in `Web/Shared/` â€”
      `QualityBadge.razor` (dot + verdict word + hover/focus tooltip with the
      plain-language reason, the reading, its meaning, "last polled" as relative
      time, response time, success rate, consecutive failures and the raw last
      error) and `WhatsThisTooltip.razor` (the "?" explainer used for
      "Communication health" and "Slave ID"). Both take the same
      `HealthSnapshotDto` so a page never has to know anything about the health
      model, and a **null snapshot renders the UNKNOWN badge** â€” "never polled"
      is a fact, not a fault, so it is never painted green or red. Badge CSS is
      in `site.css` under a dated block; the pulse animation is disabled under
      `prefers-reduced-motion`.
    - **Wiring**: `LiveTelemetryState` gained a `Health` dictionary keyed by
      `TargetExternalId` (gateway `DEV-XXXXXXXX` / sensor `S-XXXX`) plus
      `HealthFor()` and `ApplyHealthOverview()`, and it now subscribes to
      `ReceiveHealth` / `ReceiveHealthAll` on the existing hub â€” one connection
      carries telemetry AND health. `Devices.razor` and `Gateway.razor` inject the
      state, load `/api/health` on first paint, refresh it on a **throttled**
      tick (every 10th 1s tick / every 4th 4s tick â€” the hub is the primary
      source, the fetch is only a safety net against a dropped frame), subscribe
      `State.OnChange` to repaint, and **unhook in `Dispose()`** (forgetting this
      leaks a delegate for the life of the circuit and fires
      `InvokeAsync` on a dead component). `Devices.razor` also gets a
      "Link Health" roll-up card in the KPI strip, built from synthetic
      `HealthSnapshotDto`s so the header and the rows can never drift apart
      visually.
    - **Razor gotchas hit while building the UI**: a component attribute cannot
      contain a bare string literal or mixed markup â€” `Snapshot="Foo("x")"`
      fails with **RZ9986 "Component attributes do not support complex
      content"**. The fix is a `@(...)` expression for the whole value
      (`Reading="@($"{x} {y}")"`), or better, a precomputed property
      (`Snapshot="BadgeGood"`). Also `QualityBadge`'s null-snapshot explanation
      must go through a plain helper, not a leftover call.
    - **VERIFIED LIVE**: `/api/health` served 61 accumulated samples on
      `USR Gateway` (`DEV-A355817F`) = **GOOD**, latency **2.4 ms**, success
      **100%**, with the explanation string rendering correctly; the threshold
      block reaches the UI legend as `goodLatencyMs=500`,
      `badConsecutiveFailures=3`, `successRateWindow=100`, `dropWindow=5`. Web
      `/` 200, `/_framework/blazor.server.js` 200, `/account/login` 200,
      `/devices` 302 -> login (normal unauthenticated Blazor Server). Both new
      components and the client method are confirmed compiled into
      `scada_demo_test.Web.dll`, and the `.qb-tip` / `.wt-btn` CSS is served
      (site.css 78 KB). Browser click-through of the tooltip was NOT driven
      headlessly (no browser available) â€” the components are compile-verified
      and the data they bind to is verified live.
    - Builds: API **0 warnings / 0 errors**. The API must be started detached and
      with the correct working directory (see item 33) â€” `dotnet <dll> --urls ...`
      from the project dir, because `cmd` will not resolve a bare `*.exe` name.


39. **LOGIN WAS COMPLETELY BROKEN - DelegatingHandler built by hand (fixed
    2026-09-29)** - the biggest regression of the Web auth rework: **no user could
    log in at all**. `POST /account/login` returned `302` to
    `/login?error=Cannot reach API: The inner handler has not been assigned.`
    Root cause was the "build the HttpClient by hand from the circuit scope" fix
    (item 38/circuit-scoped handler): `Program.cs` did
    `new HttpClient(sp.GetRequiredService<PermissionForwardingHandler>())`.
    - **TWO separate faults, both had to be fixed**:
      (a) A `DelegatingHandler` constructs with a **null `InnerHandler`**. Only
      `IHttpClientFactory` wires one up; `new HttpClient(handler)` does NOT, so
      `base.SendAsync` throws `InvalidOperationException: The inner handler has not
      been assigned.` The terminal `HttpClientHandler` must be assigned explicitly.
      (b) The **same handler instance** was passed to TWO `HttpClient`s (the 10 s
      normal client and the 5 min bus-scan client). A `DelegatingHandler` cannot be
      shared: the first `HttpMessageInvoker` claims it and the second fails with
      `The inner handler has already been assigned`.
    - **Fix**: `Program.cs` no longer DI-registers `PermissionForwardingHandler` at
      all. `BuildApiClient(sp, timeout)` now constructs a **fresh**
      `PermissionForwardingHandler` per client, each with its own
      `InnerHandler = new HttpClientHandler { AllowAutoRedirect = false }`, still
      resolved from the circuit scope so it keeps the circuit's
      `AuthenticationStateProvider`. The `ScadaDemoTestApiClient` ctor dropped the
      unused `IHttpClientFactory` param, and the class now implements `IDisposable`
      so the two hand-built `HttpClient`/handler pairs (which the container does NOT
      track) are disposed with the scoped service instead of leaking sockets for the
      circuit's life.
    - **LESSON**: after a manual `new HttpClient(delegatingHandler)`, ALWAYS set
      `InnerHandler`. And NEVER share one handler between two clients.
    - **VERIFIED LIVE**: `POST /account/login` -> `302 /dashboard` (was the error
      redirect); `scada_api_token` (HttpOnly, JWT) + `.AspNetCore.Cookies` set;
      `/dashboard`, `/gateway`, `/devices`, `/alerts`, `/reports` all **200 with
      real data** (gateway page showed `USR Gateway`/`DEV-A355817F` + the 2 offline
      Norvi devices) when fetched with the cookie jar; Web log has **zero**
      fail/error/exception. API `/api/health` -> `good=1`, `avgLatency=2.2 ms`,
      `/api/devices` -> `3`. Web build **0 warnings / 0 errors**.
    - Running detached: API `0.0.0.0:5080` (listener PID `16828`), Web
      `localhost:5150` (listener PID `22392`).

40. **DUPLICATE SLAVE ID DETECTION + ALL METERS FOUND AUTOMATICALLY (2026-09-29)**
    â€” the user's directive was *"jitni bhi connect ho sab dikhe, operator kuch na
    tune kare, aur sahi driver mile"*. The operator has **4 meters** on one USR-W610
    bus and gave them **distinct slave IDs**; the scan now returns **4/4 with the
    correct driver, repeatedly**.
    - **LIVE GROUND TRUTH on `USR Gateway` @ 10.10.100.254:502 (after the ID
      change â€” this supersedes every older slave map in this file):**

      | Slave | Meter | Driver key | Live |
      |---|---|---|---|
      | 1 | V880BR / LUGB vortex | `VORTEX_FLOWMETER` | flow % 0 |
      | 2 | Kaifeng electromagnetic (water) | `KAIFENG_EM_FLOWMETER` | 0.636 mÂ³ |
      | 3 | Aosong AQ3485 temp/humidity | `AOSONG_AQ3485` | 29.4 Â°C / 47.1 %RH |
      | 5 | Selec RI-F200-C power | `SELEC_POWER_METER` | 0 kW / 0 kWh |

      No duplicate IDs remain on the bus. Slaves 1,2,3,5 = **4 of 4**, `unknownResponders=0`,
      `ambiguousSlaves=0`, `duplicateIdConflicts=[]`, **3/3 consecutive runs identical**.
    - **THE 741-PROBE SWEEP WAS KILLING THE GATEWAY (real hardware fault, fixed).**
      `AutoPasses` was `3`, so "Find Sensors" issued `247 x 3 = 741` probes over ~4
      minutes. That reliably left the USR-W610 serial bridge **wedged**: `ping` True,
      TCP `:502` accepts, the reachability endpoint answers `reachable=true` in ~3 ms
      (it only proves the listener is alive) â€” but **no Modbus frame ever returns**
      and every scan then reports `GATEWAY_UNREACHABLE`. The fix is always a gateway
      **power-cycle**; do not debug the scanner for this state. `AutoPasses` is now
      **`1`** â€” a full 1..247 sweep is 247 probes, still **every** address covered,
      and the bus is stressed a third as hard. Verified: 75-81 s per full scan.
    - **BUT ONE PASS FLATTENED IDENTIFICATION ACCURACY, so a targeted refine was
      added** (`AddressRefineAttempts = 2` in `ModbusScanner`). Measured failure: a
      1-pass sweep answered on all four meters but identified the AQ3485 as
      `THERMAL_MASS_FLOWMETER`, because the 2-register humidity window it keys off
      was garbled on that single attempt. Repeating all 741 probes fixed accuracy but
      wedged the gateway, so instead **only the addresses that answered are re-probed**
      and merged by the existing best-`CandidateRank`-wins rule. Cost is a handful of
      probes (4 meters x 2) instead of 494, with the accuracy of the old multi-pass
      sweep. This is the pattern to keep: **cheap targeted retries, never brute-force
      re-sweeps of the whole bus.**
    - **DUPLICATE SLAVE ID â€” new `duplicateIdConflicts` channel.** When two meters share
      one address the reply's **byte count contradicts the request**; the scanner OR-s
      that signal per address across the sweep and the conflict loop then re-reads the
      winner's own window. A confirmed conflict is **removed from `found`** (so it can
      never be selected, Select-All'd or added â€” its data is permanently wrong) and
      returned in the new `DuplicateSlaveIdConflictDto` list. `Gateway.razor` renders
      it as a red `modal-alert error` banner with the per-address message, and the
      "Detected Meters" header shows `N usable, M blocked by duplicate slave ID`.
    - **THE FALSE-POSITIVE THAT ALMOST SHIPPED â€” do not lower this threshold.** The
      first version used a **2-of-4 majority**. One full 741-probe sweep made a
      perfectly healthy **Selec at slave 5 contradict its own register map**, and the
      majority rule then **hid a healthy, correctly-identified meter from the
      operator** (run 4 of 4: `CONFLICT-s5`, the power meter vanished). A byte-count
      contradiction is **weak** evidence on this bridge â€” it garbles frames under load
      even with one healthy device. The rule now demands **unanimity**
      (`ConflictProbeThreshold = 4` of 4) before it is allowed to remove a meter, and
      the message never guesses the hidden meter's type. Measured healthy baseline
      once the bus was quiet: 30/30 clean reads on slave 3 and slave 5.
    - **A MEASUREMENT TRAP THAT PRODUCED A FAKE "COLLISION" â€” do not repeat it.** The
      first evidence for a collision was read with `FC03 @90 qty 10` at slave 1. The
      EM driver only ever issues **2-register** windows (`totalizer@90 qty2`,
      `flowRate@98 qty2`, proof `@92 qty2`); a 10-register request is unsupported, so
      the "corrupt byte count" was **my own probe artifact**, not a collision. Re-probed
      correctly, the EM answered 28/28 clean. Always probe a meter's **real** windows,
      and decode the `byteCount` **field** (`buf[8]`), not `frameLength - 8`, which is
      off by one.
    - **The Vortex is genuinely hard to see on this bus.** At slave 1 its windows
      (`@1026/@1028/@1032`) intermittently return **exception 0x04** or all zeros, and
      a bus-wide sweep of `FC04@1026` over slaves 1..60 finds data at **only slave 5** â€”
      which is the Selec's known zero-fill trap (item 32), *not* a vortex. The vortex is
      identified via its `CorroborationWindow @1067` and the narrow 2-register window,
      so it ranks correctly; a genuine duplicate ID hiding behind it cannot be typed by
      software and is therefore reported as a conflict rather than guessed.
    - **Web**: `ScadaDemoTestApiClient.DuplicateSlaveIdConflictDto` +
      `GatewayScanResultDto.DuplicateIdConflicts`, and the `Gateway.razor` banner.
    - **VERIFIED LIVE**: 3 consecutive full scans = 4/4 meters, correct driver every
      time, 0 unknown, 0 ambiguous, 0 conflicts, 75-81 s each. Web `POST /account/login`
      -> `302 /dashboard`, `/gateway` -> 200, `/_framework/blazor.server.js` -> 200,
      Web log zero fail/error/exception. API + Web builds **0 warnings / 0 errors**.
    - Temporary collision mock (`colmock.py`, 127.0.0.1:12601) and its device were
      created to force a deterministic collision, then **deleted**; DB left clean.

41. **DUPLICATE-ID VOTE GATE HARDENED + ACCURATE CONFLICT COUNT (2026-09-29)** - the
    user's rule is exact: *"jis slave ID ki kisi se bhi match nahi ho rahi (unique) wo
    hamesha found ho hi ho; jitni bhi duplicate ho wo sub hide rahein."* Both halves are
    now proven on the real bus AND on two purpose-built mocks.
    - **THE VOTE GATE (what decides "duplicate")** - a family qualifies ONLY when it
      earns `>= ConflictVoteThreshold (2)` **STRONG** votes. A vote is strong when the
      family either **won confidently** (`bestConfident`: corroborated by its own extra
      block AND reporting live data) or **proved itself** (`bestScore.Unproven == 0`).
      The DECISION is now the **VOTE GATE ONLY**: `collidingFamilies.Count >= 2`
      (>= 2 distinct families each with strong votes). A `framingConflict` byte-count
      contradiction is NO LONGER a standalone trigger - see the S5 follow-up below.
      It must NEVER consult `acceptedFamilies` either (that was an earlier false
      positive: real S2 EM was hidden).
    - **REAL-BUS FALSE POSITIVE FOUND AND FIXED (the important one)**: with the earlier
      scheme the winner always cast a token **weight-1** vote even when it was neither
      confident nor proven. On the real bus the RI-F200-C's corroboration block hiccups
      for one read, and the **AQ3485 ghost** (the meter's CT primary rating `1000,0`
      decoding to a plausible `100 %RH / 0.0 degC`) won that single pass weakly. Two such
      weak wins over the 2-pass + 2-refine sweep crossed the threshold and a **UNIQUE
      slave 5 was reported as a 2-meter conflict** (`votes=AOSONG_AQ3485=2,
      SELEC_POWER_METER=2`). Fix: a non-confident, non-proven winner casts **NO vote**;
      only strong evidence qualifies. Re-verified: real bus now returns **found=2**
      (`S2 KAIFENG_EM_FLOWMETER`, `S5 SELEC_POWER_METER`), `conflicts=0`, `ambiguous=0`.
    - **S5 FRAMING FALSE POSITIVE - PROVEN UNUSABLE, DO NOT REINTRODUCE (2026-09-29,
      supersedes every earlier framing line)**: the `framingConflict` byte-count path
      was tried as a standalone trigger, then **re-tested at 6/6 unanimity** (constants
      raised 4 -> 6 to shake off bridge noise). **It STILL false-positived on the UNIQUE
      Selec RI-F200-C in 4 of 8 consecutive real-bus runs** (`conflicts=[S5x3]`,
      `[S5x2]`, `[S5x2]`, `[S5x2]` while S5 was perfectly healthy), and hid it in
      4 more. Raising the bar cannot fix it: this meter contradicts its OWN byte count
      persistently, and the USR-W610 bridge garbles frames under sweep load on top of
      that. **Final decision: the framing path is computed for the `DROPPED` log line
      ONLY and is never the decision** - `if (collidingFamilies.Count >= 2)` and nothing
      else. The user's absolute rule (a unique slave must ALWAYS be found) outranks the
      ability to auto-spot an identical-family duplicate. **Re-verified after the final
      build: real bus 8/8 consecutive runs `found=[S2 KAIFENG_EM, S5 SELEC] conflicts=[]`
      - both meters found every single time.**
    - **CONFLICT MESSAGE SIMPLIFIED (user request 2026-09-29)**: the message is now the
      single line `"Your {n} sensors are using the same slave ID {slave}. Change all but
      one to a different slave ID, then scan again."` - no driver name, no "one reads
      as ...". `DuplicateSlaveIdConflictDto.DriverName` is `null`; `meterCount` still
      carries the accurate count (2/3/4). `Gateway.razor` renders ONLY `@c.Message` per
      conflict (the long header + the dashed "Tip" block were deleted).
    - **ACCURATE COUNT WITHOUT RE-BREAKING THE GATE**: once a duplicate is CONFIRMED, the
      message must count every real collider - not just the ones that won quiet passes.
      New per-address `acceptedFamilies` set records every distinct family that plausibly
      decoded at that address in any pass (winners AND losing candidates). `meterCount =
      acceptedFamilies[slave].Count` when `>= 2`, else `collidingFamilies.Count`.
      On the shared-3 mock the AOSONG never wins a pass (no corroboration window ->
      `Unproven=1` forever) so winner-only votes said 2; `acceptedFamilies` correctly
      reports **3** (EM + Vortex + AOSONG). It is used for the COUNT only, never the gate
      (it also admits ghosts, which is exactly what would re-trigger the S5 false
      positive).
    - **NEUTRAL LOG LINE (2026-09-29)**: the `DROPPED` warning used to end with
      `looks like {Driver}` - a driver guess that contradicted the neutral message and
      is exactly the kind of guess that made a later AI trust the wrong family. It now
      logs `meters={Count}` (the accurate collider count) and no driver name at all.
    - **`ProbeSlaveAsync` return extended 6 -> 7 elements**: `(Hit, ConnectionUnsafe,
      Confident, Rank, FramingConflict, Votes, AcceptedFamilies)`. All 4 return sites
      updated (`Array.Empty<string>()` for the empty/unknown paths; the final path returns
      `candidates.Select(DriverKey).Distinct()`). Both call sites (sweep + refine) feed
      `RecordAcceptedFamilies`. New helper `RecordAcceptedFamilies` + `acceptedFamilies`
      dict beside `familyVotes`.
    - **VERIFIED LIVE - all cases, after the final fix**:
      1. **Real bus** `USR Gateway` @10.10.100.254:502, slaves 1..10, passes=2 ->
         **8/8 consecutive runs** `found=[S2 KAIFENG_EM_FLOWMETER, S5 SELEC_POWER_METER]`,
         `conflicts=[]`, `ambiguous=[]`, `unknownResponders=0`.
      2. **Shared-3 mock** (`dupmock.py`, 127.0.0.1:12602, slave 2 serving EM + Vortex +
         AOSONG deterministically, byte-identical re-reads) -> 3/3 runs `responding=1,
         found=0, conflicts=1`, **count=3** "Your 3 sensors are using the same slave ID 2..."
      3. **Unique zero-fill mock** (`uniqmock.py`, same port, single Selec-like meter at
         slave 5 answering EVERY FC04 read with a byte-count and zeros + its own real
         block) -> `responding=1, found=1 (SELEC_POWER_METER), conflicts=0` - the S5
         false-positive scenario reproduced in isolation and proven dead.
      4. **Same-driver mock** (`samemock.py`, 127.0.0.1:12603, slave 7 with TWO identical
         AOSONG meters; the first 4 reads answer cleanly so the meter can be identified,
         every later answer contradicts the requested byte count) -> 4/4 runs
         `found=0, conflicts=1` "Your 2 sensors are using the same slave ID 7...". This is
         the **proof that the 6/6 framing signal itself works** - it is only the real
         meter + real bridge combination that makes it unsafe (see the S5 section).
    - **ADD-TIME UNIQUENESS IS ENFORCED (server-side, any driver)** - the operator's
      second rule *"a gateway me koi bhi slave ID dobara add nahi ho sakti"*. Live-verified
      in `SensorsController.Create` (~line 380): with a `KAIFENG_EM_FLOWMETER` already on
      slave 1 of the device, creating a `SELEC_POWER_METER` on the SAME slave 1 returns
      **HTTP 400** "Modbus slave address 1 is already used by another sensor on this
      device. Each slave needs a unique address on the RS-485 bus." The check compares
      `SlaveAddress` only, so it is driver-agnostic. A client-side guard mirrors it on
      the Save button. This is what protects the operator from the same-driver duplicate
      that the scan can no longer auto-spot.
    - **Mock gotcha (cost a cycle)**: a deterministic shared-address mock must answer the
      **exact register layout** the driver parses - AQ3485 `@0 qty2` is **2xUInt16**
      (humidity/10, temperature/10 -> 4 bytes), NOT 2 floats (8 bytes). A byte-count that
      disagrees makes the scanner treat the reply as a framing conflict. Also a
      round-robin-per-request mock FAILS (responding=0): `ConfirmsAsync` demands
      **byte-identical** repeat reads, so per-window deterministic serving is required.
      A `samemock.py` counter MUST be **per-slave**, not global: the sweep hammers empty
      addresses first, so a global budget is exhausted before slave 7 is ever probed
      (responding=0). `CreateDeviceRequest` `hardwareType` is the string `"USR-W610"`
      (not `USR_W610`) or the API 400s, and a **second mock device on the same
      `ipAddress` returns 409** (duplicate-IP guard) - delete the old one or reuse it.
    - Build: Infrastructure + API + Web **0 warnings / 0 errors**. The vote-gate-only
      decision, the 6/6 framing constants (log-only) and the simplified message are in
      this build. Temp mock devices deleted, mocks killed, DB clean (3 real devices,
      0 sensors). **RUNNING NOW** (WMI-detached): API `0.0.0.0:5080`,
      Web `localhost:5150`.
    - **STILL OPEN (product, honestly stated)**: an **identical-family** duplicate
      (two same-model meters on one ID) is NOT auto-detected, because the only signal
      that can see it - frame collision - false-positives on the real bus (measured 4/8
      at 6/6, above). Two identical meters answering one address are also fundamentally
      indistinguishable from one when their replies are not garbled, so no scan can
      count them. Coverage today: **distinct-family duplicates are hidden with an
      accurate count**, and **every duplicate is blocked at add time** by the
      driver-agnostic 400 guard. An identical-family duplicate is the operator's eyes.

42. **EM FLOWMETER DRIVER AUDIT + NEGATIVE-ZERO FIX (2026-09-29)** - the operator reported
    the electromagnetic flow meter "not performing like the others". Audited end to end
    (library -> driver -> raw bus -> scan). The driver is **correct**; one real (small)
    decode defect was found and fixed.
    - **LIVE BUS MAP - IT CHANGED, use this, not any older map**: slave **2** =
      `KAIFENG_EM_FLOWMETER`, slave **3** = `AOSONG_AQ3485` (the operator MOVED the
      humidity from 2 to 3), slave **5** = `SELEC_POWER_METER`. Slaves 1, 4, 6-247 empty.
      **Consequence for the "duplicate" question**: while both the EM and the humidity
      sat on slave 2 they WERE a genuine duplicate and were correctly hidden. After the
      operator moved the humidity to 3, slave 2 holds only the EM, so it is a UNIQUE
      meter and is CORRECTLY shown. Its appearing in the results is the right outcome,
      not a bug - do not "fix" it by suppressing the driver.
    - **LIBRARY CROSS-CHECK (no change needed)**: `C:\library_own_dx\lab\ModbusMeterLibs`
      `src\Modbus.Electromagnetic\ElectromagneticDriver.cs` + `ElectromagneticSettings.cs`
      say FC **0x03**, `StartAddress 90`, `TotalizerOffset 0` (regs 90-91),
      `FlowRateOffset 8` (regs 98-99), `WordOrder.HighWordFirst`, `InterReadDelayMs 50`.
      `ElectromagneticFlowmeterDriver` is a faithful 1:1 port. The reference project
      reads the two values as two separate 2-register requests, which the driver also
      does (two `ReadWindows`).
    - **RAW BUS PROBE** (`%TEMP%\opencode\dupmock\probe.py`, fresh connection per read,
      the same way the scanner reads): slave 2 `FC03 @90 qty2` = regs `[16162, 51900]`
      -> **0.6359 m3** totalizer; `@92 qty2` = `[15230, 44099]` -> 0.003886 (non-zero,
      so the `CorroborationWindow` genuinely PROVES the signature); `@98 qty2` =
      `[32768, 0]`; `@90 qty10` -> byteCount 20 with a coherent block; `FC04 @1026` ->
      `EXCEPTION 0x02` (so the Vortex cannot fake it). Slave 3 `FC03 @0 qty2` =
      `[500, 291]` -> 50.0 %RH / 29.1 C, and `FC04 @0` -> `EXCEPTION 0x81` (illegal
      function), which is why the AQ3485 is FC03-only here.
    - **THE DEFECT: NEGATIVE ZERO.** `[32768, 0]` is float `0x80000000` = **-0.0**, a
      signed zero, because the meter's flow register carries the sign bit while the flow
      is genuinely zero. It compares equal to `0.0` everywhere, but it stringifies as
      `-0` / `-0.000` and leaked into the dashboard, the charts and the CSV/PDF report
      exports - i.e. it genuinely did not "perform like the others". Fixed once for every
      driver in `Domain/Drivers/ModbusValueCodec.ToFloat32`: the exact negative zero is
      folded to positive zero (`return value == 0d ? 0d : value;`). A genuinely NEGATIVE
      reading (reverse flow) is deliberately left untouched - only `-0.0` is remapped.
      Verified live: the EM now reports `InstantaneousFlowRate = 0` instead of `-0`.
    - **VERIFIED, every scan mode**: narrow `1..10 passes=2` **3/3** and the full
      `1..247 passes=0` "Find Sensors" sweep (**267 addresses, 3 responding**) both
      return `found=3, conflicts=0, ambiguous=0, unknown=0` -
      `S2 KAIFENG_EM_FLOWMETER` (0 m3/h, 0.636 m3), `S3 AOSONG_AQ3485` (29.1 C, 50 %RH),
      `S5 SELEC_POWER_METER` (0 kW, 0 kWh). A **raw narrow probe is not enough** to call
      this meter good: the first symptom the operator reported was a drop that only
      appears under the full 247-address sweep, so always re-test the full sweep.
    - Builds: Domain + Infrastructure + API **0 warnings / 0 errors**. `AGENTS.md` is the
      only place the old slave-2/3/5 map should be corrected from now on.

43. **MASKED-DUPLICATE DETECTION + SLABE-5 GHOST FIX - COMPLETE, VERIFIED (2026-09-30)**
    - item 42 concluded "slave 2 holds only the EM", which was WRONG: the operator
      confirmed **slave 2 = EM + vortex, slave 3 = Aosong, slave 5 = Selec** (4 meters,
      2 sharing address 2). The correct scan result is `found=2` + one hidden conflict
      for slave 2 with `meterCount=2`. **That result is now produced, repeatedly.**
    - **Why the vote gate alone could never do it**: the EM is the faster responder, so
      it masks the vortex on ~19 of 20 reads. The vortex therefore never *wins* a pass
      and `familyVotes` only ever saw one family. `acceptedFamilies` is not usable either
      (it admits the AQ3485/thermal ghosts), and the winner's-own-window framing path
      is the one that false-positived 4/8 on the UNIQUE Selec.
    - **THE SIGNAL - foreign-window reply RATE, not framing.** A block the address's own
      meter does not serve is answered by exactly one of three regimes, and only the
      middle one means a masked second device:
      | address | AOSONG@0 | EM@90 | SELEC@42 | VORTEX@1026 |
      |---|---|---|---|---|
      | **2 (EM+vortex)** | 77/80 = 96% | - | 0/80 = 0% | **3/80 = 4%** |
      | 3 unique Aosong | - | 0/80 | 0/80 | 0/80 |
      | 5 unique Selec | 34/40 = 85% | 34/40 = 85% | - | 30/40 = 75% |
      `0%` = nobody serves it, `~100%` = the address's own meter (or a zero-filler like
      the RI-F200-C) owns it, **a few percent = one device is racing another**. Witness
      band is deliberately narrow: `answered > 0 && answered * 4 <= sampled` (<= 25%).
      Both healthy regimes sit far outside it, which is what makes this safe where the
      framing vote was not. A malformed frame on a foreign window is also an immediate
      witness (that is how the same-driver mock is caught).
    - **Narrow windows only** (`RegisterQuantity == 2`), winner's own window skipped.
      Wide reads garble on a SINGLE device - the healthy Selec at slave 5 gave 12 clean +
      8 corrupt on the 12-register thermal window - so wide windows are never used here.
      This is the trap that killed the earlier "any mixed outcome" idea.
    - **Implementation** (`ModbusScanner.cs`): new
      `HasForeignWindowCollisionAsync(...)` + constants `ForeignCollisionProbeBudget
      = 300`, `ForeignCollisionReadsPerConnection = 15`, `ForeignCollisionDelayMs = 20`,
      `ForeignCollisionGiveUpSamples = 40`, `DegeneracyProofAttempts = 4`.
      Round-robins foreign narrow windows, reuses one TCP connection for 15 reads
      (measured: fresh connection ~92 ms/read vs reused ~50 ms, identical leak rate).
      A cheap early-exit marks a window useless once it is mostly answering, so a clean
      address pays ~40 samples per window instead of the full budget. Decision:
      `if (collidingFamilies.Count >= 2 || foreignCollision)`, with
      `if (meterCount < 2) meterCount = 2;`. The `DROPPED` log gained `foreign={Foreign}`.
    - **THE SLAVE-5 GHOST (the second bug, also fixed)**: slave 5 was misidentified as
      `AOSONG_AQ3485` in ~1 of 3 runs. Cause: the RI-F200-C genuinely reads 0 kW / 0 kWh
      (CT not connected), so the degeneracy guard rejected it, and the AQ3488 GHOST -
      the meter's CT primary rating `(1000, 0)` decoded as a plausible `HumidityRH=100,
      TemperatureC=0` - won that pass. The `Unproven` tie-break never ran because the
      driver was REJECTED before comparison. **Fix = retry, NOT a looser guard**: the
      guard now re-reads the driver's own `CorroborationWindow` up to 4 more times before
      rejecting (`DegeneracyProofAttempts`). A direct measurement on the real bus showed
      that proof read returns live data **60 of 60** attempts, so the old 2-attempt
      loop was simply losing a genuine match to a transport hiccup. The retry is only
      ever reached by a driver that is about to be thrown away, so a healthy sweep costs
      nothing. Do NOT reintroduce a general zero-tolerant guard - that is what let a
      humidity sensor win as a flowmeter back in item 31.
    - **Web**: `ScadaDemoTestApiClient.ScanGatewayBusAsync` budget adds
      `collisionCheckMsPerMeter (15000) * maxMetersOnOneBus (8)`, otherwise a legitimate
      scan is cancelled client-side. Verified adequate: full 247-address scan = 155 s
      vs a 409 s budget.
    - **VERIFIED LIVE - 6/6 real-bus runs identical**: narrow `1..10 passes=1`
      (108-122 s each) and the full `1..247 passes=0` sweep (155 s, 267 addresses,
      3 responding, gateway still pingable afterwards) all return
      `found=2 [S3 AOSONG_AQ3485, S5 SELEC_POWER_METER] + conflicts=1 [S2, meters=2]`,
      `ambiguous=0`, `unknown=0`. The log line is the evidence:
      `slave 2 foreign-window collision - VORTEX_FLOWMETER @1026 answered only 3-4 of
      100 reads`, with slaves 3 and 5 clean.
    - **VERIFIED on the purpose-built mocks (all three)**: `uniqmock` (unique
      zero-fill Selec at slave 5) -> `found=1, conflicts=0` **no false positive**, and
      only 15 s because the give-up rule short-circuits it. `dupmock` (shared-3) ->
      `conflicts=1, meters=3` via the VOTE gate, with the foreign check correctly clean
      (that mock answers foreign windows at 40/40). `samemock` (two identical AOSONG on
      slave 7) -> `conflicts=1, meters=2` via the new foreign trigger
      (`foreign=True`, malformed frame) - i.e. same-driver duplicates are now caught
      too, which is BETTER than item 41 claimed was possible.
    - Measurement scripts kept as evidence in `%TEMP%\opencode\dupmock`: `proof.py`
      (this bug: proof read 60/60, q2 vs q4 both fine), `matrix.py` (per-driver window
      matrix - the run that proved the 12-reg width trap), `validate_rule.py` (0 corrupt
      in 600 unique reads vs 3 in 180 on the collided address), `drain.py` (the gateway
      forwards only the WINNING frame - a second frame appears in 1/25 reads),
      `reuse.py` (reuse is 2x faster, same leak rate), `structural.py`, `vortexhunt.py`
      (the vortex signature is served NOWHERE on the bus - only the Selec's zero-fill
      at slave 5 - which is why it had never been identified at all), `disambiguate.py`,
      `collide.py`, `framing.py`.
    - Builds: Infrastructure + API + Web **0 warnings / 0 errors** (Web exe must be
      stopped first - MSB3021/MSB3027 file lock). DB left CLEAN: 3 real devices, 0
      sensors, all temp mock devices deleted, 0 mock listeners.
    - **RUNNING NOW** (WMI-detached, `ASPNETCORE_ENVIRONMENT=Development`, **no Debug
      logging**): API `0.0.0.0:5080` (PID 13616, log `%TEMP%\opencode\api-run.log`),
      Web `localhost:5150` (PID 14288, log `%TEMP%\opencode\web-run.log`).
      `/_framework/blazor.server.js` 200, `POST /account/login` -> 302 `/dashboard`,
      `/gateway` + `/dashboard` 200 with the cookie jar, `/api/health` -> `good=1,
      avgLatency 2.2 ms`, `/api/sensors/libraries` -> 5 drivers, startup line lists all
      5, zero fail/error in either log. Debug logging can be re-enabled with
      `set Logging__LogLevel__Default=Debug` when per-address scoring evidence is
      needed.

44. **QUICK-SCAN SPEED + HONEST TIME ESTIMATE (2026-09-30)** - the scan was correct but
    took ~188 s while the UI promised "~15 s", and the promise was wrong by 10x.
    **The bus was re-mapped mid-session by the operator (see the header), so the live
    slave numbers in this item supersede every older excerpt in this file.**
    - **WHERE THE TIME ACTUALLY WENT (measured, not guessed).** A `Scan: TIMING` debug
      line in `ModbusScanner` (`sweepMs` / `checkMs` / `totalMs`) settled it: the
      address sweep was only ~34-50 s and the **duplicate-verification pass was
      ~108-131 s**, i.e. **~72% of every scan**. The old UI formula only counted
      silent addresses, which is why it was off by an order of magnitude.
    - **A RESPONDING address is not one probe.** Every installed driver window is tried,
      then re-read to confirm byte-identical replies, then the corroboration block and
      extra windows: **~3.2 s per responding address per pass** (5 drivers, so a
      zero-filler like the RI-F200-C - which answers every block - is the most
      expensive address on the bus).
    - **FIX 1 - the delay is only needed on a window JUMP, not between repeats.**
      `ForeignCollisionDelayMs` 20 -> 0 for a repeated block, with the turn-around kept
      as `ForeignCollisionWindowJumpMs = 20` when the register block changes. Proven
      safe by `dupmock/delaytest.py`: 100 back-to-back samples of one window at
      60 / 20 / 0 ms gave **125 / 83 / 65 ms per sample with byte-identical
      classification** on all three test addresses (masked 100%, unique-empty 0%, unique
      zero-fill ~100%). The leak RATE is a property of two transceivers racing to answer
      the same request, not of how fast the master asks, so the gap cannot move it.
      The delay must NOT be dropped on a jump: that desyncs the shared session and
      manufactures the very malformed frame the check treats as a witness, i.e. a false
      positive on a healthy bus.
    - **FIX 2 - sample each foreign block CONSECUTIVELY instead of round-robin.**
      `perWindow = ForeignCollisionProbeBudget / foreign.Length` samples back to back per
      block. **The sample count per block - and therefore detection confidence - is
      completely unchanged**; what changes is that a block jump happens 3 times instead
      of ~200. This is the change worth keeping: it is free speed, not a trade.
    - **FIX 3 - a fresh TCP connection per register block**, then reuse it for that
      block's whole run (only 2 extra connections per address now that samples are
      consecutive). This matches the USR-W610's one reliable behaviour, and measured
      **150 ms/read when one session was driven across blocks vs 65-70 ms on a fresh
      connection** - the reused session was burning its time on stale replies.
      `ForeignCollisionReadsPerConnection` 15 -> 40.
    - **FIX 4 - early witness exit** (`ForeignCollisionEarlyWitnessSamples = 25`): a
      masked device answers only a few percent, so the verdict locks long before the
      block's share is spent, and a positive detection now returns in ~25 reads instead
      of ~300. The 25-read floor is what stops one lucky frame on a healthy block from
      hiding a uniquely addressed meter.
    - **HONEST ESTIMATE** (`Gateway.razor`): `ScanEstimateSeconds` is now built from the
      measured constants - silent address 235 ms, responding address 3.2 s x passes,
      verification ~36 s **per meter found** - with `ScanExpectedMeters = 3`. The
      scanning banner says outright that the check grows with each meter found and the
      estimate can run slightly over or under. Passes are `ScanPassesQuick = 3` and
      `ScanPassesFull = 1` constants, no longer a hard-coded `3`.
    - **RESULT: ~188 s -> ~165 s, `found` byte-identical.** Sample counts per block, the
      4x witness threshold, the give-up rule and the decision are all unchanged, so
      nothing about *what is found* moved. The remaining time is the irreducible
      statistical cost of proving a few-percent leak - that is the real trade, and it was
      **not** taken silently.
    - **REGRESSION PROVEN ON ALL THREE MOCKS after the rework** (this is what makes the
      speed acceptable):
      - `uniqmock` (unique zero-fill Selec at slave 5) -> `found=1, conflicts=0` - **no
        false positive**, the S5 scenario from item 43 still dead.
      - `dupmock` (3 families share slave 2) -> **`conflicts=1, meters=3`, 2/2 runs**.
      - `samemock` (2 identical AOSONG on slave 7) -> `conflicts=1, meters=2` on the
        first run via the malformed-frame witness. **The second run returned
        `responding=0`** - the mock's per-slave frame budget is exhausted once its own
        driver has been identified, so the sweep saw nothing at all. That is the known
        mock limitation from item 41, not a scanner regression: the witness path itself
        fired and was logged.
    - **A measurement trap that cost a cycle, do not repeat**: `uniqmock.py` and
      `dupmock.py` **both hardcode PORT 12602**, and `uniqmock` wins the bind, so
      starting both and assuming the second one is live silently tests the wrong mock -
      it reported `found=1 conflicts=0`, which is uniqmock's CORRECT result and looked
      exactly like a regression. **Kill the listener on 12602 and start the intended
      mock before every mock scan**, and assert the port is listening first. Also note
      `CreateDeviceRequest`'s field is `Name` (not `deviceName`), and a second mock
      device on `127.0.0.1` is rejected **409** by the duplicate-IP guard - delete the
      first mock device before creating the next.
    - **`PowderShell` display trap**: `Invoke-RestMethod` on a JSON array plus PS 5.1
      member enumeration makes `foreach` print every field of every element on one line
      and makes `.Count` report the FIRST element's field count, so a 3-device list shows
      `devices: 1` and `libraries: 1`. The data is fine. Count with `@(...).Length`, and
      prefer the API's own array shape.
    - Builds: API + Web **0 warnings / 0 errors**. `/_framework/blazor.server.js` 200,
      `/api/health` -> `good=1`, `libraries` -> 5 drivers, zero fail/error in either log.
      **NOTE (2026-10-01 re-measurement): the live bus map in this item-45 paragraph
      is now SUPERSEDED by the measured map in the recovery banner above - the real
      bus is S1=Kaifeng EM, S2=Vortex, S3=Aosong, S4=Selec, NOT 1=Aosong/2=Vortex/3=EM/4=Selec.
      Trust the banner, not this older line.**
    - **RUNNING NOW** (WMI-detached, `ASPNETCORE_ENVIRONMENT=Development`, **no Debug
      logging**): API `0.0.0.0:5080` (PID 14760), Web `localhost:5150` (PID 6288).
      DB clean: 3 devices (gateway online, 2 Norvi push-idle offline), 0 sensors, all
      temp mock devices deleted, no mock listeners left. Debug timing evidence needs
      `set Logging__LogLevel__Default=Debug`.
    - **STILL OPEN / honest limitation**: the FULL scan's verification cost is **linear in
      the number of meters found** (~36 s each), so a bus with many meters still feels
      slow there. That is a deliberate trade: it is the only way to see a meter HIDING
      behind another on one slave ID. **This is now the operator's explicit choice** -
      see item 45; Quick skips it and Full keeps it.

45. **QUICK SCAN = 165 s -> ~42 s, DEEP DUPLICATE CHECK NOW AN EXPLICIT MODE (2026-09-30)**
    - the user's directive was exact: *"sirf quick scan karo sahi se"*, and the previous
      Quick took **165 s** while the label on screen promised far less. Root cause was
      already known from item 44: the duplicate-verification pass was ~72% of the run.
    - **THE FIX IS A REQUEST FLAG, NOT A WEAKER DETECTOR.**
      `SmartScanRequest.DeepDuplicateCheck` (default **`true`**), mirrored in the API's
      `ScanBusRequest` (`bool?` -> `req?.DeepDuplicateCheck ?? true`) and in the Web
      client. `ModbusScanner` gates the two EXPENSIVE deep checks on it - the
      winner's-own-window framing re-read and `HasForeignWindowCollisionAsync`. The
      free **vote gate** (>= 2 distinct families winning/proving themselves at one
      address) still runs in Quick. The Web Quick button sends `false`
      (`deepDuplicateCheck: !quick`); "Find Sensors" sends `true`.
    - **THE TRADEOFF IS SMALLER THAN IT LOOKS - measured, not assumed.** On the
      `dupmock` mock (3 families sharing slave 2) Quick with `deepDuplicateCheck:false`
      still returned **`conflicts=1, meters=3`**, because that mock's families all win
      passes and the vote gate is part of the sweep. What Quick gives up is only the
      **masked** duplicate, where a fast meter hides a slow one that never wins a pass
      - the user's original real-bus case. The Quick button's tooltip says this
      explicitly rather than promising full parity.
    - **Foreign-window read timeout `120 ms`** (`ForeignCollisionReadTimeoutMs`, applied
      as `Math.Min(probeTimeoutMs, ForeignCollisionReadTimeoutMs)`). Measured on the
      real bus with `dupmock/latency.py`: served frames **57-71 ms**, silent blocks
      **196-204 ms**, so a 200 ms probe wasted ~80 ms on every silent foreign window
      and a 120 ms cap keeps ~1.7x headroom over the slowest served read. This only
      affects the deep path.
    - **MEASURED RESULT (real `USR Gateway` @ 10.10.100.254, 4 meters on distinct IDs)**:
      | mode | wall clock | check | found |
      |---|---|---|---|
      | Quick `deep=false` | **39 / 47 / 45 / 42 s** (4 runs) | `checkMs=0` | **4/4** every run |
      | Quick `deep=true` | 153 s | `checkMs=106 s` | 4/4 |
      The `Scan: TIMING` line is the proof the gate works: `checkMs=0` on every Quick run
      versus `sweepMs=45.8 s / checkMs=106.7 s` with it on.
      Live bus map in these runs: **S1 Aosong, S2 Vortex, S3 Kaifeng EM, S4 Selec**.
    - **REGRESSION: all three purpose-built mocks re-verified on the real binary, deep
      mode ON** (the 120 ms cap could have broken the witness reads):
      - `uniqmock` (unique zero-fill Selec at slave 5) -> `found=1`, **`conflicts=0`**
        - the S5 false-positive scenario is still dead. 12 s.
      - `dupmock` (3 families share slave 2) -> **`conflicts=1, meters=3`**, 2/2 runs.
      - `samemock` (two IDENTICAL AQ3485 on slave 7) -> **`conflicts=1, meters=2`** via
        the malformed-frame witness. Note `samemock.py` listens on **12603**, while
        `uniqmock.py`/`dupmock.py` share **12602** - check the port before scanning or
        you silently test the wrong mock.
      Mock devices deleted and all mock listeners killed afterwards; DB left with the
      3 real devices and 0 sensors.
    - **THE PROGRESS LABEL NO LONGER OUTLIVES ITS OWN PROMISE.** Two fixes:
      (a) the estimate is now an explicit **upper bound** built from measured worst
      cases (silent address 260 ms, responding address 5 s, 8 meters assumed) instead
      of a best guess that was wrong by 10x; (b) new `ScanOverEstimate` drops the quoted
      number the moment real time passes it and swaps in "running longer than usual -
      this is not a hang", because a counter climbing past its own estimate makes a
      working scan look frozen and invites a page reload.
    - **RAZOR GOTCHAS that cost real time here - do not repeat.** Writing the progress
      sentence inline in the markup threw `RZ1000 "unterminated string literal"` plus a
      cascade of ~30 bogus `RZ9980 "unclosed tag 'string' / 'Dictionary' / 'int'"` errors
      on types far below the edit, because the sentence's apostrophe (in "the gateway's")
      flipped Razor out of markup. Two things are needed, not one: keep the **prose in C#
      properties** (no apostrophes/dashes in markup), and write it as **plain string
      concatenation, NOT interpolated strings** - Razor counts braces inside string
      literals while tracking the `@code` block's nesting, and a `$"{cond ? "" : "..."}"`
      in the middle of the text ended the block early. The misleading part is that the
      first error line is NOT the cause; always fix the topmost `RZ1000` and rebuild
      before reading the cascade.
    - **Two measurement traps hit again, both already documented - they still cost
      time.** (a) The scan DTO field is **`driverKey`/`displayName`**, not
      `detectedDriverKey`; a PowerShell regex/property guess reported "S1 missing" when
      `found` in fact held **all 4** meters. Parse the JSON properly
      (`Invoke-RestMethod` + `.found`) before believing a miss. (b) The API response
      field for a detected count is `respondingSlaves`, and `found` is a list - count it
      with `@($r.found).Length`.
    - Builds: API + Web **0 warnings / 0 errors**. `/_framework/blazor.server.js` 200,
      `POST /account/login` -> 302 `/dashboard`, `/api/health` -> `good=1`,
      `avgLatency 2.3 ms`, `/api/sensors/libraries` -> 5 drivers, startup line lists all
      5. API log has `warn=0` / `exception=0` (the `fail`/`error` substring hits are EF
      column names `ConsecutiveFailures` / `LastError` in the health-snapshot SQL, not
      faults). Web log zero fail/error/exception.
    - **RUNNING NOW** (WMI-detached, `ASPNETCORE_ENVIRONMENT=Development`, **no Debug
      logging**): API `0.0.0.0:5080` (listener PID 4544), Web `localhost:5150`
      (listener PID 2696), logs `%TEMP%\opencode\api-run.log` + `web-run.log`.
      Re-enable per-address scoring evidence with `set Logging__LogLevel__Default=Debug`.
    - **STILL OPEN (honest)**: one Quick run out of four identified the Selec at slave 4
      as `AOSONG_AQ3485`. That is the **pre-existing** item-43 ghost - the RI-F200-C
      reads a genuine `0 kW / 0 kWh` because its CT is not connected, so the degeneracy
      guard needs its corroboration block to survive; when that one read fails, the
      meter's CT primary rating decodes as a plausible `100 %RH / 0.0 degC`. Not caused
      by this item, and not fixable by weakening the guard (that is what once let a
      humidity sensor win as a flowmeter). A cheap improvement worth doing if the
      operator asks: retry the Selec proof block more than `DegeneracyProofAttempts=4`.



## Known context (verified facts)

- Firmware POST JSON shape:
  `{"tank_id":<modbus_slave_addr>,"tankReadings":[{"metric":"Flowrate","value":12.34,"unit":"Nm3h"},...]}`
  (`json1` in firmware). **`tank_id` = Modbus slave address, not device id.**
  No `deviceId`/`externalId` in the payload â†’ source-IP attribution.
- Firmware posts to `http://192.168.100.128:5080/api/tank-reading`
  (`send_data.h`). Legacy Laravel URL in `global.cpp` is unused. For live
  attribution the device's DB `IpAddress` MUST equal the firmware's actual source
  IP (e.g. `my_norvi` = `192.168.1.150`).
- Firmware "failure" branch sends `"--"` strings as values â€” the API tolerates them
  (values are parsed permissively; `"--"`/garbage â†’ skipped reading, never a 400).
  The firmware-side fix (send a number/null) remains optional polish.
- Devices in DB (verified 2026-09-29): **3 devices** â€” `USR Gateway` (USR-W610 @
  10.10.100.254:502, **online**) plus `Norvi 56` @192.168.100.56 and
  `Norvi 243` @192.168.100.243 (both Norvi ESP32, both **offline** and push-idle,
  which is the correct data-driven state â€” they are wired off the network).
  The gateway carries **4 meters on distinct slave IDs** (see item 40 for the full
  map and live values). The gateway is on ETHERNET (not WiFi) and the DB is local, so
  the polling delay is just the configured `PollIntervalSeconds`. Users can set this
  per-sensor in the Add-Sensor/Edit-Parameters modals; 1 s is the fastest allowed.
    - Connections: API `0.0.0.0:5080`, Web `localhost:5150`, login
        `superadmin@alamiot.com` / `SuperAdmin@123`. **DB = local SQL Server LocalDB
        `scada_db_iiot`** (instance `(localdb)\MSSQLLocalDB`; SQL Server 17.0.4025.3).
        Revert path for Supabase: `DB_LOCAL_SWITCH_CHANGELOG.md`.
    - **LOGIN ROUTES (do not confuse)**: the login PAGE is `GET /login`
        (`Pages/Login.razor`), and it contains an HTML form that POSTs to
        **`/account/login`** (a minimal-API endpoint in `Web/Program.cs`, not a
        Blazor route). `GET /account/login` is therefore a 404 shell â€” never test
        login by fetching it. A protected page returns `302 -> /login`.
- **ALERT-RULE GOTCHA (2026-09-22)**: right before this session the USER created 3
  rules via the Web UI â€” all on `deviceExternalId: "ALL"` with **legacy metric names**
  `Pressure` >8.5 (Critical), `FlowRate` <10 (Warning), `FlowRate` >250 (Critical).
  The alert engine matches ONLY real driver columns (`InstantaneousFlowRate`,
  `AccumulatedTotalizer`, `TemperatureC`, `HumidityRH`), so **these 3 rules will never
  fire** and the in-app notification toast won't trigger from them. The Alerts UI form
  itself only offers real metrics â€” the user's stale-browser/legacy rules predate that.
  To make notifications work: delete/recreate those rules from the Alerts UI with the
  real metric dropdown (e.g. Flow Rate â†’ matches InstantaneousFlowRate). Notification
  pipeline IS live now: poller â†’ `AlertService.EvaluateReadingAsync` per selected
  column â†’ `AlertIncidents` row (2-min debounce) â†’ SignalR `ReceiveAlertIncident` â†’
  MainLayout toast stack. W610 offline â†’ nothing triggers until the gateway returns.

## NEXT STEPS (if any further work is wanted)

1. **LocalDB hardening (DONE 2026-09-22)**: verified end-to-end on LocalDB with a
   local mock Modbus gateway (127.0.0.1:12502, `%TEMP%\opencode\modbusmock`):
   login â†’ add gateway `E2E-LOCALDB` (USR-W610) â†’ add 2 sensors (Kaifeng slave 1
   + AOSONG slave 2) â†’ poller streamed real values (Kaifeng 2.5 mÂ³/h / 123.4 mÂ³,
   AOSONG 25 Â°C / 42 %RH, 18 rows each in the T-SQL telemetry tables) â†’ device +
   sensors ONLINE â†’ killed mock â†’ watchdog flipped everything OFFLINE after the
   60s timeout (exactly as real hardware behaves) â†’ deleted test device (telemetry
   tables dropped). Rollup worker no-ops harmlessly (legacy `SensorReadings`,
   0 rows). DB left clean.
2. **Real-hardware validation**: connect the actual Norvi on the network where its
   DB-registered IP matches the push source, confirm the dashboard flips it ONLINE
   with live values within 4s (Web polling) and OFFLINE ~30s after pushes stop.
3. **Real-bus scan validation (DONE 2026-09-29 â€” supersedes the 09-28 note)**: the real
   USR-W610 bus (`USR Gateway` @ 10.10.100.254:502) now finds **all 4 attached meters
   with the correct driver, 3/3 identical full scans** (75-81 s each). The old "20 ms
   probe may miss 9600-baud meters" note is obsolete: the probe timeout is a fixed
   `const 200` in `Gateway.razor` and "Find Sensors" sweeps the whole bus 1..247 in a
   **single** pass (247 probes) plus a **targeted re-probe of only the addresses that
   answered** (`AddressRefineAttempts = 2`). Do NOT restore multi-pass over the whole
   bus â€” 247x3 = 741 probes wedges the W610 bridge and needs a power-cycle (item 40).
   Slave map is 1=vortex, 2=EM, 3=Aosong, 5=Selec; 4, 6-247 are EMPTY. If a new meter
   is connected, scan and expect `ambiguousSlaves` to stay 0 on a well-behaved bus.
4. ~~`TelemetryIngestService` removal~~ **DONE 2026-09-22**: legacy
   `TelemetryIngestService` / `ITelemetryIngestService` / `TelemetryMessageDto`
   (old external-id tank-reading path, unused since the TankReading rewrite)
   removed â€” files deleted + DI registration dropped. `LiveReadingDto` still lives
   in the same DTO file (poller/broadcaster use it). The live push path is
   `PushTelemetryIngestService` (TelemetryController).
5. **Firmware "--" â†’ "null" (DONE on firmware side 2026-09-22)**: all 4 failure
   branches of `firmware/Norvi-ESP32/lib/sensor_data/sensor_data.cpp` now send
   `"value":null` instead of `"value":--` (which produced INVALID JSON). API
   `ToNullableDouble` returns null for `JsonValueKind.Null` â†’ skipped reading,
   never a 400. Success branches unchanged. PlatformIO not on this machine â€”
   flash via `pio run -t upload` from a machine with PlatformIO.
6. **Gateway tab live check (DONE 2026-09-21)**: API + Web restarted (latest PIDs
   7740 / 11760). Verified by curl: API `/api/devices` 200 (1 device),
   `/api/sensors/libraries` 200 (3 drivers), login `superadmin@alamiot.com` OK,
   Web `/`, `/devices`, `/gateway` all 302 â†’ `/account/login` (expected for
   unauthenticated Blazor Server). Final browser pass (login â†’ sidebar Devices
   parent group â†’ `/gateway` Step 1) remains for the human operator.

## Metric alias map (firmware metric name â†’ telemetry column)

| Firmware/source metric | Driver column |
|---|---|
| `Temperature`, `Temp`, `TemperatureC` | `TemperatureC` |
| `Humidity`, `HumidityRH` | `HumidityRH` |
| `Flowrate`, `Flow`, `FlowRate`, `InstantaneousFlowRate`, `Nm3h` | `InstantaneousFlowRate` |
| `Totalizer`, `AccumulatedTotalizer`, `Total` | `AccumulatedTotalizer` |

## Rules for the next AI

- After ANY state change (implemented step, new finding), update this section:
  move items from IN PROGRESS -> DONE, refresh NEXT STEPS, update timestamp.
- Build only the changed project(s) for speed (`dotnet build` on the API or Web
  project directly, never the whole `.slnx`).
- No commits unless the user explicitly asks.

## 46. AI STUDIO + VISUAL STUDIO DUAL-ENVIRONMENT CHECKPOINT (2026-10-02)

- **Pure .NET 10 Stack Preserved**: Zero React/Vite/Node UI files. The solution runs
  natively on **.NET 10 (`net10.0`)** across `Domain`, `Application`, `Infrastructure`,
  `API`, `Web` (Blazor Server), and `Maui`.
- **Restored Untracked Driver Files (`src/scada_demo_test.Domain/Drivers/`)**:
  - `ModbusValueCodec.cs` (IEEE-754 float32 high/low-word-first + `-0.0` -> `0.0` fold)
  - `VortexFlowmeterDriver.cs` (`VORTEX_FLOWMETER`, FC04 @1026/1028/1032, proof @1067)
  - `SelecPowerMeterDriver.cs` (`SELEC_POWER_METER`, FC04 low-word-first @42/0, proof @64)
- **Dual-Environment Database Configuration**:
  - **Windows PC / Visual Studio (`OperatingSystem.IsWindows() == true`)**: Uses
    `UseSqlServer(DefaultConnection)` pointing to local SSMS LocalDB
    (`Server=(localdb)\MSSQLLocalDB;Database=scada_db_iiot;...`), `MigrateAsync()`,
    T-SQL `DynamicSchemaInitializer`, and T-SQL `SensorTelemetryRepository`.
  - **Linux Cloud / AI Studio Container (`OperatingSystem.IsWindows() == false`)**:
    Automatically uses `UseSqlite("Data Source=scada_db_iiot.sqlite")` with
    `EnsureCreatedAsync()` + `IdentitySeeder.SeedAsync()` (`superadmin@alamiot.com` /
    `SuperAdmin@123`) and SQLite-compatible DDL/queries in `SensorTelemetryRepository`.
- **Web Auth & Handler Fixes (`src/scada_demo_test.Web`)**:
  - `Program.cs` builds each `HttpClient` with a fresh `PermissionForwardingHandler`
    whose `InnerHandler = new HttpClientHandler { AllowAutoRedirect = false }` is
    explicitly assigned, and forwards `scada_api_token` from `IHttpContextAccessor`
    during initial SSR/prerender requests.
  - Cross-site iframe cookies (`SameSite=None; Secure`) enabled automatically on
    non-Windows container runs while keeping standard localhost cookie behavior on Windows.
- **Hybrid Workflow**: Code edits made in AI Studio sync via GitHub (`git pull`) to
  the operator's local Windows PC (Visual Studio + SSMS LocalDB + physical WiFi/Ethernet
  hardware at `10.10.100.254:502`), while also compiling and running live inside AI Studio
  on ports `3000` (Web) and `5080` (API).

## 47. SCADA ENGINE CORE ARCHITECTURE & 3-BUCKET DUAL-GUARD PIPELINE (2026-10-02)

- **Domain Models & Envelopes (`ScadaEngine.Core.Models`)**:
  - `DeviceProfileType` Enum (`AosongAQ3485`, `V880BRVortex`, `KaifengThermal`, `KaifengElectromagnetic`, `SelecPower`, `Unknown`).
  - `ModbusDevicePacket` (The Strict Envelope): contains `byte SlaveId`, `DeviceProfileType ProfileType`, `string ModelName`, `byte[] RawPayload`, `DateTime Timestamp`, `string DriverKey`, `ushort StartRegister`, `ushort RegisterQuantity`, and `RawHex`.
  - `RawScanResponse`: contains `byte SlaveId`, `byte[] Payload`, `bool IsSuccess`, `string? ErrorMessage`, `DeviceProfileType? HintedProfile`, `byte FunctionCode`, `ushort StartRegister`, `ushort RegisterQuantity`.
  - `ScanResultSummary`: holds 3 distinct result buckets:
    1. `AlreadyInSystemDevices` (`List<byte>` & `List<AlreadyInSystemDeviceInfo>`): existing registered IDs.
    2. `BlockedScanDuplicates` (`List<byte>` & `List<BlockedScanDuplicateInfo>`): conflicting IDs on the same bus session.
    3. `UniqueFoundDevices` (`List<ModbusDevicePacket>`): verified identity envelopes in the "Found Box".
- **Checkpost Router & Signature Engine (`CheckpostRouter`)**:
  - Zero routing DB dependency: purely in-memory using `FrozenDictionary` and `ReadOnlySpan<byte>` with microsecond execution latency.
  - Strict byte boundary & payload inspection:
    - Length 4 bytes -> `AosongAQ3485` (16-bit Ints)
    - Length >= 8 bytes -> `V880BRVortex` / `KaifengElectromagnetic` / `SelecPower` (32-bit IEEE Floats)
    - Length 6 bytes -> `KaifengThermal`
  - Rejects noise/corrupted frames upfront, returning a fully tagged `ModbusDevicePacket`.
- **Dual-Guard Scanner Service (`ModbusScannerService`)**:
  - Pre-Scan: loads database-registered slave IDs into `HashSet<byte> _dbRegisteredSlaveIds`.
  - **GUARD 1 (DB Existence Guard)**: If `_dbRegisteredSlaveIds.Contains(slaveId)`, routes directly to `AlreadyInSystemDevices` (UI: *"Already added in system. Change hardware ID"*). Bypasses raw bucket.
  - **GUARD 2 (Current Scan Duplicate Guard - BEFORE Found Box)**: Groups raw responses in `RawBucket` by `SlaveId`. If `Count() > 1`, blocks all duplicates and moves `SlaveId` to `BlockedScanDuplicates` (UI: *"Bus conflict: Same ID responded multiple times"*).
  - **FOUND BOX TRANSFER**: If `Count() == 1`, passes payload to `CheckpostRouter.InspectAndTag()` to produce `ModbusDevicePacket`, which is transferred to `UniqueFoundDevices` ("Found Box").
- **Sensor Driver Dispatcher (`SensorDriverDispatcher`)**:
  - In-memory O(1) dictionary routing to designated calculation logic:
    `AQ3485Ghar()`, `V880BRGhar()`, `KaifengThermalGhar()`, `ElectromagneticGhar()`, `SelecPowerGhar()`.
  - Integrated into `SensorDriverCatalog.DispatchPacket(ModbusDevicePacket)`.
- **Modbus Polling Engine Alignment (`ModbusPollingHostedService`)**:
  - Adheres strictly to Architectural Rule #1 (No blind processing) and Rule #2 (In-memory O(1) routing).
  - Polled raw frames are inspected and tagged into `ModbusDevicePacket` envelopes before dispatching to engineering calculations.
- **Hardware Safety & Half-Duplex RS-485 Sequential Probing (`ModbusScanner`)**:
  - Fixed concurrent `Task.WhenAll(probe3, probe4)` on serial bridge: probes are strictly sequential with turn-around delays. If FC03 succeeds, FC04 probe is skipped immediately.
- **Web UI & Performance Overhaul (`ScadaDemoTestApiClient` & `Gateway.razor`)**:
  - Eliminated slow sequential internet roundtrips to Firebase Realtime DB from `ScadaDemoTestApiClient`.
  - Switched `Users`, `Roles`, `Tanks`, `Sites`, and `AuditLogs` to direct local ASP.NET Core API endpoints (`/api/users`, `/api/roles`, `/api/tanks`, `/api/sites`, `/api/audit-logs`) with graceful fallback.
  - Reduced `Devices.razor` polling timer from 1000ms to 3000ms, eliminating UI lag.
  - Added visual 3-bucket presentation in `Gateway.razor`:
    - **Bucket 1 (Already In System)**: Amber card with slave ID and sensor name.
    - **Bucket 2 (Blocked Scan Duplicates)**: Red alert card with bus collision warning and count.
    - **Bucket 3 (Unique Found Devices / Found Box)**: Green verified identity cards with ProfileType badge, ModelName, Hex Payload preview, and the restored **"+ Add Selected Sensor(s)"** button!
- **Build & Verification Status**:
  - .NET 10 compilation: **0 errors, 0 warnings**.
  - Dev server running cleanly on port 3000 (Web) and port 5080 (API). Both returning HTTP 200.

## 48. NAMED SENSOR DUPLICATE CONFLICT RESOLUTION ENGINE (2026-10-02)

- **Problem Solved**: When RS-485 sensors collide on the same Slave ID (e.g., Slave 2), the operator sitting in the office previously only saw a generic error saying "Same ID responded multiple times". They did not know WHICH physical meters were fighting for that ID, making it impossible to instruct field technicians.
- **Microsecond Signature Disambiguation & Name Attribution**:
  - During the presence and signature trial phases of `ModbusScanner`, all responding register profiles are captured per address.
  - When a duplicate collision is declared (`colliding.Count >= 2` or foreign window collision), the scanner extracts the distinct driver keys (`AOSONG_AQ3485`, `VORTEX_FLOWMETER`, `KAIFENG_EM_FLOWMETER`, `SELEC_POWER_METER`, etc.) and tags each raw response with its friendly display name (`ModelName`).
  - `ModbusScannerService` groups the responses in `RawBucket`, extracts the unique colliding names, and constructs actionable operator instructions:
    *Example:* `"Slave 2 Collision: Meter [Kaifeng Electromagnetic Flowmeter] aur [V880BR Vortex Steam/Gas Flowmeter] dono same Slave ID 2 use kar rahe hain! Worker ko bolein ke in mein se kisi ek meter ka Modbus ID change kare."*
  - Populates `CollidingMeterNames` in `DuplicateSlaveIdConflictDto` and `DuplicateIdConflictDto`.
- **Operator-Grade Web UI in `Gateway.razor` (Bucket 2)**:
  - Renders visual badges for each colliding meter:
    `[Kaifeng Electromagnetic Flowmeter]` ⚡ CONFLICT ⚡ `[V880BR Vortex Flowmeter]` on `Slave ID 2`.
  - Displays a high-contrast instructions banner for the on-site technician: *"Instructions for Worker: Ask the on-site technician to change the Modbus Slave ID of either meter to an unused ID, then run Scan again."*
  - Allows an operator with zero on-site visibility to effortlessly resolve hardware conflicts remotely.



