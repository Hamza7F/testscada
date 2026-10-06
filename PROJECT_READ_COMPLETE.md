# PROJECT_READ_COMPLETE.md (MAUI excluded)

Scope: Full read of all source files except MAUI project. Total: 144 files, ~21,382 lines.

## Directory breakdown

- src/scada_demo_test.Domain: 44 files (~2,696 lines)
- src/scada_demo_test.Application: 18 files (~869 lines)
- src/scada_demo_test.Infrastructure: 25 files (~4,004 lines)
- src/scada_demo_test.API: 18 files (~2,124 lines)
- src/scada_demo_test.Web: 32 files (~7,967 lines)
- tests/ScadaEngine.Tests: 4 files (~371 lines)

## Critical findings verified

1) Drivers (4 only)
- SensorDriverCatalog has: AosongAQ3485Driver, ElectromagneticFlowmeterDriver, VortexFlowmeterDriver, SelecPowerMeterDriver.
- KAIFENG_FLOWMETER (thermal mass) is NOT present.
- Vortex FC04 windows 1026/2,1032/2 proof 1067/2; Selec FC04 low-word-first 42/2,58/2 proof 64/10.

2) ScadaEngine (envelope + fingerprint + dispatcher)
- ModbusDevicePacket, DeviceProfileType, RawScanResponse, ScanResultSummary.
- CheckpostRouter: in-memory O(1), signature trial, InspectAndTag.
- CheckpostFingerprintEngine: evidence required; matches.Length==1; hint cannot substitute; function/register context checked.
- SensorDriverDispatcher: dispatch by packet profile.
- FingerprintTests: every driver identifies 1-247; boundaries/immutability/reserved/duplicates/registered precedence.

3) Infrastructure Modbus/Scanner
- ModbusTcpMaster: TCP session, exception classification, MBAP.
- ModbusScanner: 3-bucket dual-guard aware; Quick passes=1 deep=false end=20, Full passes=0 deep=true end=247.
- CandidateRank.CompareTo: Unproven ASC first, then other.Width.CompareTo(Width) so WIDER width ranks higher (descending). Live/Extra descending, Order last.
- ModbusScannerService: Guard 1 -> Bucket 1 (AlreadyInSystem), Guard 2 -> Bucket 2 (Blocked duplicates), else -> Router/Dispatcher -> Bucket 3 (Found).
- ScannerTests cover framing, timeouts, every-address scan, registered not probed, duplicates blocked, partial rejected, cancellation, lease.

4) API
- Program.cs: Windows -> SqlServer+MigrateAsync; non-Windows -> SQLite+EnsureCreatedAsync; background DB retry; policies Tab*/Action*; 4 drivers registered; hosted services; hub /hubs/telemetry.
- DevicesController: scan with DeepDuplicateCheck/Passes/Start/End, /{id}/reachability.
- SensorsController: libraries from catalog (4 drivers).
- No /api/health endpoint.

5) Web
- Program.cs: fresh PermissionForwardingHandler per HttpClient with explicit InnerHandler.
- ScadaDemoTestApiClient: IDisposable (2 HttpClients); API-first with some Firebase fallback.
- Gateway.razor: 3-bucket UI (AlreadyInSystem, Blocked duplicates with colliding names, Found). Quick vs Full as above.
- Devices.razor: CRUD + always-visible child sensors.

## Discrepancies vs AGENTS.md
- Drivers: 4 only (no KAIFENG_FLOWMETER).
- No /api/health, QualityBadge, WhatsThisTooltip.
- Scanner ranking: width compared DESC (wider-first), not narrowest-first.

