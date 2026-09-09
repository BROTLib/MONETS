# Reading live diagnostics from the running controller via ADS

How to check the real, running MONETS controller's state from this dev machine when something looks wrong — used during the 2026-09-09 `bInterrupted` deadlock investigation (`../plans/2026-09-09-static-review-and-live-ads-findings.md`).

## Connection

- Target: `CX-9BBD41`, AmsNetId `10.1.180.77.1.1`, TC3 PLC runtime port `851`.
- This machine already has a static AMS route to it (`C:\TwinCAT\3.1\Target\StaticRoutes.xml`) and a full TwinCAT 3 install (`TcSysSrv`, AMS router, `TcXaeShell`), so no route setup is needed — just connect.
- No `pyads`/Python on this machine. Script from PowerShell by loading the .NET ADS API directly from the GAC:

```powershell
Add-Type -Path "C:\TwinCAT\AdsApi\.NET\v4.0.30319\TwinCAT.Ads.dll"
$client = New-Object TwinCAT.Ads.TcAdsClient
$client.Connect("10.1.180.77.1.1", 851)
$client.ReadState()          # confirms AdsState/DeviceState, e.g. Run
$client.ReadAny($client.CreateVariableHandle("MAIN.SomeSymbol"), [bool])   # or [uint32], [double], etc.
$client.Dispose()
```

**This is read-only** (`ReadState`, `ReadAny`/`CreateVariableHandle`) — no writes. Only write to the live controller (e.g. forcing a variable) with explicit sign-off first; it's a live telescope.

## What to read when an error is reported

TcXaeShell's online Error List (the `Severity/Code/Description/.../Line` table, e.g. `'TCNC' (500): 'Elevation' ... error 0x4260`) is **not persisted anywhere** — confirmed by checking `C:\TwinCAT\3.1\Boot\LoggedEvents.db` (a SQLite file) after the 2026-09-09 incident: its last-write timestamp predated the incident by weeks, so it hadn't captured those messages. They're live ADS Logger (AMS port `Logger` / 100) output, visible only while TcXaeShell is connected online at the moment they occur. **If you want the exact human-readable message text, you must have TcXaeShell open and connected at the time, or paste it from there afterward** — there is currently no verified programmatic way to pull historical Error List text from this session (a live ADS Logger notification subscription on port 100 might work for *future* messages, but this was not tested — don't assume it works without verifying first).

What *can* be read after the fact, reliably, via the method above — because the ST code stores the last error's structured data in plain output variables:

| Symbol | Meaning |
|---|---|
| `MAIN.TelescopeControl.bError` | any axis (Elevation/Azimuth/Derotator/Focus) currently has an error |
| `MAIN.TelescopeControl.nErrorID` | the error ID copied from whichever axis is currently faulted (`FB_MonetTelescopeControl.TcPOU:59-79`) |
| `MAIN.ElevationControl.bError` / `.nErrorID` | Elevation axis error flag / ID directly |
| `MAIN.AzimuthControl.bError` / `.nErrorID` | Azimuth axis, same |
| `MAIN.DerotatorControl.bError` / `.nErrorID` | Derotator axis, same |
| `MAIN.TelescopeControl.bInterrupted` | latched-interrupt state — see `telescopecontrol-command-arbitration.md` |
| `MAIN.TelescopeControl.bReady` / `.bBusy` / `.bStopped` | readiness/motion state |
| `MAIN.ElevationControl.bEnable` / `MAIN.AzimuthControl.bEnable` / `MAIN.DerotatorControl.bEnable` | whether `_PowerOn()` has actually re-armed each axis |

Caveat: `bError`/`nErrorID` reflect the *current* scan, not history — if the fault has already self-cleared (as happened on 2026-09-09; `bError` read `False` well after the incident), these will read clean even though something did go wrong recently. They're most useful read *during* or immediately after a live incident, not as a retrospective log.

## Worked example (2026-09-09 incident)

Read live, well after the fault had already self-cleared:

```
MAIN.TelescopeControl.bInterrupted = True     <- still latched
MAIN.TelescopeControl.bReady       = False
MAIN.TelescopeControl.bBusy        = False
MAIN.TelescopeControl.bStopped     = True
MAIN.TelescopeControl.bError       = False    <- fault itself had already cleared
MAIN.ElevationControl.bEnable      = False    <- but never got re-armed
MAIN.AzimuthControl.bEnable        = False
MAIN.DerotatorControl.bEnable      = True
```

The *exact* NC fault codes (`0x4260`, `0x4358`, `0x4257`) and their text came only from Tim pasting the TcXaeShell Error List at the time — they were not recoverable via ADS after the fact.
