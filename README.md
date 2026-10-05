# test_today_one Demo → Real Hardware Guide

Everything is currently set to **DEMO mode** (6 simulated flowmeters, no
hardware needed). This file lists every place you touch when you're ready
to connect the real Norvi/ESP32 gateway with 1 real flowmeter.

## Part A - What's already done, no action needed

- `send_data_to_backend()` in the firmware already builds the correct JSON
  and does the HTTP POST - **you do not need to write any new firmware code**.
- The `.NET` backend already has a `TelemetryController` that accepts that
  exact JSON shape and auto-registers a new device the first time it sees it -
  **no manual device registration needed**.
- Modbus reading, register addresses, display, WiFi reconnect logic - all
  unchanged, all already correct for your Kaifeng flowmeter.

## Part B - What YOU change when hardware is ready

### 1. Firmware: `firmware/Norvi-ESP32/lib/network/network.cpp`
Find `init_Wifi()` near the top. Change:
```cpp
String ssid = "Marriott_Admin";     // <-- your WiFi name
String password = "KMHIT.123";      // <-- your WiFi password
```
Must be the **same WiFi network** your PC (running scada_demo_test.API) is on.

### 2. Firmware: `firmware/Norvi-ESP32/lib/send_data/send_data.cpp`
Find `send_data_to_backend()`. Change the IP in this line to your PC's LAN IP
(run `ipconfig` on the PC, use the IPv4 Address):
```cpp
String targetUrl = "http://192.168.1.15:5080/api/telemetry/tank-reading"; // <-- your PC's IP
```

### 3. Firmware: `firmware/Norvi-ESP32/src/main.cpp`
Already trimmed to **1 active meter** (slave_id 1) inside `Task2code`. The
other 13 lines are commented - uncomment one per meter as you wire more up.
Nothing else to change here for the first meter.

### 4. Backend: `src/scada_demo_test.API/Program.cs`
Comment out this ONE line (turns the demo simulator off so the dashboard
only shows your real meter):
```csharp
// builder.Services.AddHostedService<SimulatedTelemetryHostedService>();
```
That's it - `TelemetryController` (the real endpoint) is already always live,
nothing to uncomment there.

### 5. Windows Firewall (run once, as Administrator)
```
netsh advfirewall firewall add rule name="test_today_one API" dir=in action=allow protocol=TCP localport=5080
```
Needed because the Norvi reaches your PC over WiFi, not localhost.

## Part C - Running it

1. `docker compose up -d timescaledb` *(or your LocalDB - already configured)*
2. `dotnet run` from `src/scada_demo_test.API` (now bound to `0.0.0.0:5080` so
   it's reachable from WiFi, not just from the PC itself)
3. `dotnet run` from `src/scada_demo_test.Web`
4. Flash the updated firmware to the Norvi, power it on
5. Watch the Norvi's Serial Monitor - you should see
   `[send_data] Success. Code: 200`
6. Open the web dashboard - your real flowmeter card should appear
   automatically (named `norvi-slave-1`) with live data

## Device ID reference (for when you add more meters later)

| slave_id | Function used | Becomes device ID |
|---|---|---|
| 1 | read_kaifeng_IEMFL_data | `norvi-slave-1` |
| 2, 3 | read_kaifeng_tg4stm | `norvi-slave-2`, `norvi-slave-3` |
| 4, 6, 11, 12 | read_kaifeng_TMGFL_data | `norvi-slave-4`, etc. |
| 5, 7, 8, 9, 10, 13, 14 | read_kaifeng_IEMFL_data | `norvi-slave-5`, etc. |

You can rename these later in the Devices page (currently the name defaults
to the device ID on first auto-registration).

## Data storage: tiered rollup / compression pipeline

Raw telemetry (as often as every 1 second, per device, per metric) is written to
`SensorReadings`. That table would grow unbounded, so a background service
(`RollupCompressionHostedService`, always running in `scada_demo_test.API`) continuously
compresses old data into three coarser tiers instead of keeping it all at full
resolution forever:

```
Raw (seconds)  --older than 3h-->  Hourly  --older than 7d-->  Daily  --older than 60d-->  Monthly (kept)
```

- **FlowRate / Pressure / other instantaneous metrics** are compressed with a
  **time-weighted average** (trapezoidal integration between samples), so uneven
  sampling gaps don't skew the number.
- **Totalizer** (and anything else marked cumulative in
  `scada_demo_test.Domain.Constants.MetricCatalog`) is compressed with **Last/First delta
  logic** that tolerates a hardware counter reset (a value drop is treated as the
  counter restarting from ~0, never as negative consumption).
- Every stage is **idempotent and fault-isolated per (device, metric, time bucket)**:
  a bucket is only inserted if it doesn't already exist, and source rows for a bucket
  are only deleted *after* the compressed row above it is committed. If one bucket
  fails (bad data, a transient DB error, etc.) it's logged and simply retried on the
  next tick - it can never crash the service or lose the underlying data, and it
  never blocks other devices/metrics from compressing normally.
- DB writes on the hot paths (`SensorReadingRepository`, `RollupRepository`) are
  wrapped in a shared Polly retry policy (`scada_demo_test.Infrastructure.Resilience.EfResilience`)
  so a one-off transient SQL error becomes a short retry instead of a dropped reading.

### Custom date-range reports

`GET /api/reports/{externalId}/history?metric=FlowRate&from=...&to=...` (and the PDF
version at `/range-pdf`) accepts **any** range - a minute, a day, a year - and
automatically reads from whichever tier still covers it (Raw/Hourly/Daily/Monthly),
so the caller never needs to know the retention policy. This powers the new
"Custom date-range report" section on the Reports page (quick presets + a chart +
a table + PDF download).

### Adding this to an existing database

This project didn't have any EF migrations checked in yet (RBAC tables included), so
after pulling this update, run once:

```
dotnet ef migrations add AddRollupTables -p src/scada_demo_test.Infrastructure -s src/scada_demo_test.API
dotnet ef database update -p src/scada_demo_test.Infrastructure -s src/scada_demo_test.API
```
