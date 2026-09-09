# 2026-09-09 — Static review + live ADS investigation

## Starting point

A prior Cowork session (cloud-only, no shell access) did a static, non-compiling read-through of `MAIN.TcPOU`, `FB_TelescopeAuxiliary.TcPOU`, `FB_WeatherCheck.TcPOU`, and `E_ModeLanguage.TcDUT`, and left a handover doc (`monets-handover.md`, repo root) with five unconfirmed findings, blocked mainly by `brotlib`/`astrobrot`/`halfbrot` being present in this checkout only as compiled `.library` blobs (no ST source).

## What this session added

1. **Located the sibling source repos** at `../BROTLib`, `../HalfBROT`, `../AstroBROT`, `../MONETcommon` (siblings of `MONETS` under `C:\Users\MONET\Documents`), unblocking the two findings that needed library internals.
2. **Finding #1 (MQTT watchdog, commented out in `MAIN.TcPOU`) — confirmed safe to re-enable.** `FB_Comm_MQTT.TcPOU` (BROTLib) already has the agreed `bConnected`-every-cycle fix plus fallback-to-reconnect on disconnect (state 3). `FB_Comm_MQTT_Influx` inherits it unchanged.
3. **Finding #2 (no-op cabinet/pendant reset) — root-caused, not just "missing a call".** Traced every FB reachable from `MAIN`'s reset block:
   - `FB_MonetSafetyHandling` has no manual reset (`outErrAck` is driven only by its own internal startup sequence) — correctly excluded from the fix, by design for a safety circuit.
   - `ElevationControl`/`AzimuthControl`/`DerotatorControl`/`FocusControl` (via `FB_BaseAxis`) and `HydraulicsControl` all expose a working `.Reset()`.
   - `FB_MonetCoverControl.Reset()` (MONETcommon) is buggy: sets `Reset := TRUE` (its own return value) instead of `bReset := TRUE`, so calling it does nothing.
4. **Connected via ADS (read-only) to the live controller** — `CX-9BBD41` (confirmed by the user to be the real device, at `192.168.127.251` / AmsNetId `10.1.180.77.1.1`), reachable from this dev machine which already has TwinCAT 3 + a static route to it. Verified this by reading `MAIN.fbComm.bConnected`, `MAIN.mqttWatchdog.Q`, `MAIN.error`, etc. — all read-only via `TwinCAT.Ads.dll` (GAC) loaded from PowerShell (no `pyads`/Python on this machine).
5. **User hypothesis: "`bInterrupted` in fbTelescope is triggered and not cleared."** Read live: `MAIN.TelescopeControl.bInterrupted = True`, `bReady = False`, `bBusy = False`, `bStopped = True`, no command flags active. Confirmed the hypothesis and traced the full deadlock mechanism — see `../design/telescopecontrol-command-arbitration.md` and [ADR 0001](../adrs/0001-telescopecontrol-reset-must-be-driven-by-main.md).
6. **Filed [BROTLib/MONETS#3](https://github.com/BROTLib/MONETS/issues/3)** covering the `bInterrupted` deadlock, the no-op reset root cause, and the `FB_MonetCoverControl.Reset()` bug, with the related dead-code findings (#1 watchdog, #4 weather shutdown, #3 Focus error, #5 WeatherCheck `bError`) noted as background.

## Root trigger identified: 2026-09-09 00:54:22 NC axis fault

Tim provided the TwinCAT error list for the actual incident that left the telescope in the deadlocked state observed live above:

```
00:54:22.678  'PlcTask' (350): Derotator Axis Error: 4358
00:54:22.690  'TCNC' (500): 'Elevation' (Axis-ID: 2, Grp-ID: 2): The axis or a coupled slave axis has lost its
              controller enable signal while executing a command => error 0x4260
              (StateDWord: 0x210411f, CoupleState: 0, ActPos: 54.936104, ActVelo: -0.014041)
00:54:22.690  'TCNC' (500): 'Azimuth' (Axis-ID: 1, Grp-ID: 1): The axis or a coupled slave axis has lost its
              controller enable signal while executing a command => error 0x4260
              (StateDWord: 0x210411f, CoupleState: 0, ActPos: 209.643517, ActVelo: -0.013624)
00:54:22.670  'TCNC' (500): 'Derotator' (Axis-ID: 3): Motion commands are not allowed for external setpoint
              generated axis (protected mode, check the axis parameter 'allow motion commands to external
              setpoint axis') (CmdType: 4096, Error: 0x4257)
00:54:22.670  'TCNC' (500): 'Derotator' (Axis-ID: 3, Grp-ID: 3): Group function is rejected with error-code
              0x4257 or the function is not supported!
00:54:22.672  'TCNC' (500): 'Derotator' (Axis-ID: 3): The axis has no 'Feed Forward Permission' while executing
              external setpoint generation => error 0x4358 (FeedEnablePlus=0; SetDir=1.000000, SetVelo=0.000000;
              VeloDir=0.000000, PosDir=1.000000)
```

Reconstructed causal chain:

1. `Derotator` runs continuously in **external setpoint generation** mode while tracking (its position is driven by `F_DerotatorPosition2(...)`, not point-to-point moves — see `fDerotatorCalc` in `FB_MonetTelescopeControl.TcPOU`). It lost **Feed Forward Permission** (`FeedEnablePlus=0`) at 00:54:22.672 → error `0x4358`.
2. Because Derotator was in that protected external-setpoint mode, any motion command directed at it in the same window was rejected outright (`0x4257`, "Motion commands are not allowed for external setpoint generated axis").
3. `Elevation` and `Azimuth` — coupled to Derotator as part of the same motion group — lost their controller enable signal (`0x4260`) 12-20ms later while a command was executing.
4. This set `fbDerotator.bError`/`fbElevation.bError`/`fbAzimuth.bError`, which set `FB_MonetTelescopeControl.bError`, which (per `FB_MonetTelescopeControl.TcPOU:93-95`) force-disabled `fbElevation.Enable`/`fbAzimuth.Enable`/`fbDerotator.Enable`.
5. Whatever command (`goto`/`slew`/`track`) was active at that moment was interrupted mid-flight, latching `bInterrupted := TRUE`.
6. The NC-level axis faults were evidently cleared or self-recovered afterward (`bError` read `False` live), and Derotator's `Enable` came back `True` — but Elevation/Azimuth's `Enable` never did, because of the `bInterrupted`/`_PowerOn()` deadlock in [ADR 0001](../adrs/0001-telescopecontrol-reset-must-be-driven-by-main.md). The dead cabinet/pendant reset (finding #2) meant nobody could break the cycle afterward.

**Open question, not yet investigated:** why did Derotator lose Feed Forward Permission in the first place? That's an NC/axis-parameter-level question (`allow motion commands to external setpoint axis`, feed-forward config) rather than something visible in the ST source — would need TwinCAT XAE Shell's NC configuration view or the relevant axis parameter list, not just the `.tsproj` text.

## Resolution (manual, same day)

Tim manually forced `MAIN.TelescopeControl.bInterrupted := FALSE` and parked the telescope successfully. Re-read live afterward: `bInterrupted = False`, `bReady = True`, `Elevation`/`Azimuth`/`Derotator` all `bEnable = True`, `bError = False`, park in progress (`bBusy = True`). This confirms the deadlock diagnosis directly — clearing `bInterrupted` was sufficient to unblock `_PowerOn()`, re-enable the axes, and let the park command execute. No code changes were needed to recover this specific incident; the manual clear is the same recovery `TelescopeControl.bReset` would have performed automatically had the reset button been wired (ADR 0001).

## Not yet done

- The code fix in ADR 0001 (wiring `MAIN`'s reset block to `TelescopeControl.bReset` + `CoverControl.Reset()`/`HydraulicsControl.Reset()`) has **not** been applied — Tim asked to hold off. The manual ADS write above was a one-off operational recovery, not a substitute for the fix; the same deadlock will recur on the next axis fault that interrupts an in-flight command.
- `FB_MonetSafetyHandling`'s hardware-driven `outErrAck` sequence and `FB_TelescopeAuxiliary`/`E_ModeLanguage` were not re-examined this session (nothing new since the original handover).
- Root cause of the Derotator Feed Forward Permission loss (the actual trigger, see below) is still open.

## Live ADS access notes (for next time)

- Route already exists: `CX-9BBD41` in `C:\TwinCAT\3.1\Target\StaticRoutes.xml`, AmsNetId `10.1.180.77.1.1`, port 851 (TC3 PLC runtime, first instance).
- No `pyads` available; scripted via PowerShell + `Add-Type -Path "C:\TwinCAT\AdsApi\.NET\v4.0.30319\TwinCAT.Ads.dll"` — `TcAdsClient.Connect(netId, port)`, `ReadAny(CreateVariableHandle(symbolPath), type)`.
- `git`/`gh` were not installed on this machine at the start of this session; both installed via `winget install --id Git.Git -e --silent` and `winget install --id GitHub.cli -e`. `gh auth login` requires interactive user action (browser/token) — cannot be done non-interactively.
