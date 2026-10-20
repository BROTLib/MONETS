# MONET/S: project-level hardware test plan (MONETS PLC)

Date: 2026-10-20. Target: the MONETS TwinCAT 3 project (`MONETS/MONETSRuntime`), branch `develop`.

This plan covers behaviour that is **configured or implemented in the MONETS project itself**: the
parameters passed in `MAIN.TcPOU`, the call order and error aggregation in `MAIN`, the MONET/S
auxiliary temperature sensors (`FB_MonetsTelescopeAuxiliary`), the VISU bindings and the MQTT
endpoint configuration. Library internals (MONETcommon, HalfBROT, BROTLib, AstroBROT) have their own
plans; where a test here depends on library logic it says so. Tests already done in the general
telescope test plan are referenced by its IDs (T0-T15) and not repeated.

## Safety rules (hard)

- **The roof is never opened.** No step opens the roof, no step runs a night with the roof open.
  Any item that needs an open roof is listed under "Left open". `RoofControl` symbols are read only.
- Forcing variables, online changes, activations, SoE commands, Wake&Shake and any XAE action are
  done by the human operator only and are marked **USER (XAE)**.
- The remote operator (**REMOTE**) only uses ADS (AmsNetId `10.1.180.77.1.1`, port `851`, symbols by
  full path) and MQTT (broker `192.168.127.10:1883`, command topic `MONETS/Telescope/SET`, influx
  line protocol, e.g. `command azimuth=150,elevation=50,slew=1`; verbs `power`, `park`, `gohome`,
  `track`, `slew`, `stop`, `reset`, `focus`).
- ADS writes are only made to command inputs (e.g. `bReset`, `fPosition`). Physical inputs (`AT%I*`
  variables such as `InOilLevel`, `IbOilLow`, `tempSensorM1`) are overwritten by the I/O image every
  cycle, so faking them requires a **force in XAE (USER)**.
- Every test has an abort criterion. General abort for all tests: unexpected motion, a drive error
  that does not clear with `reset`, oil pressure lost while an axis is enabled, any E-Stop event, loss
  of ADS or MQTT for more than 60 s during motion. Abort action: MQTT `command stop=1`, then
  `command park=1`; if that fails, USER presses the E-Stop / switches the key to local.
- Before leaving any test: all forces released (XAE force list empty), telescope parked
  (`MAIN.TelescopeControl.bIsParked = TRUE`), no latched errors.

## Prerequisites

1. General telescope test plan T0 (backup, rollback point) and T1 (activation of current build) done
   for the build under test; record the version from `Global_Version` and the git commit.
2. USER has XAE online on the target, with the force list empty.
3. REMOTE has an ADS client and an MQTT client subscribed to `MONETS/Telemetry` and `MONETS/Log`,
   and read access to the Influx database fed from MQTT.
4. Telescope parked, covers closed, roof closed, cabinet key in remote (`CabinetControl.KeyLocal =
   FALSE`, i.e. `MAIN.eTelescopeMode = automatic`).
5. Someone with a camera view or on site for all tests marked "motion" (see risk column).

## Reference values from `MAIN.TcPOU`

| Item | Value | Where defined |
|---|---|---|
| Park position (az / el / derot / focus) | 175 / 60 / 0 / 36.0 | `telescopeConfig` (MAIN) |
| Home position (az / el / derot) | 175 / 40 / 0 | `telescopeConfig` (MAIN) |
| Azimuth soft limits | -274.0 .. 270.0 | `AzimuthControl(fMinPosition, fMaxPosition)` (MAIN) |
| Derotator soft limits | -90.0 .. 360.0 | `DerotatorControl(...)` (MAIN) |
| Elevation soft limits | 0.0 .. 90.0, hard-coded `LIMIT` | HalfBROT `FB_ElevationControl`, not configurable from MAIN |
| Elevation hardware limits | drive digital inputs bit 0 (pos), bit 1 (neg) | HalfBROT `FB_ElevationControl` |
| Focus soft limits | 0 .. 100 (library defaults, not overridden in MAIN) | HalfBROT `FB_FocusControl` |
| Focus hardware limits | `IbLimitNear`, `IbLimitFar` (far = homing cam) | HalfBROT `FB_FocusControl` |
| Focus homing / speed / brake delay | 92.02 / 0.2 / T#30S | MAIN |
| Oil pan sensor scaling | raw 3500 -> 0 %, raw 10200 -> 100 %, no clipping (`cut := FALSE`) | MAIN + HalfBROT |
| Suction start / stop | level > 50.0 % or pan float switch / level < 0.0 % | MAIN (`fMaxPanPercent := 50.0`, was 56.0) |
| Hydraulics timers | pressure watchdog 30 s, suction watchdog 15 s, pressure-loss brake close 2 s, brake open delay 3 s, auto-off 300 s after brake closed | HalfBROT |
| Azimuth limit switch input to hydraulics | `bAzimutLimitSwitch` **not wired in MAIN** | see MS-06 |
| Telemetry period (MAIN) | 5 s (`commTimer`) | MAIN |
| MQTT watchdog | park + close roof after 30 s without broker | MONETcommon `FB_MonetMqttWatchdog` |

## Test overview

Risk: L = read only / no motion, M = motion within normal range or a forced input with no motion,
H = motion towards a limit or a forced safety-relevant input.

| ID | Topic | Who executes | Risk |
|---|---|---|---|
| MS-01 | Oil level reading and scaling | REMOTE | L |
| MS-02 | Oil level sensor broken / out of range | USER (XAE) + REMOTE | M |
| MS-03 | Oil low / reserve / hot / cold flags and their effect on the telescope | USER (XAE) + REMOTE | H |
| MS-04 | Oil pressure loss while the brake is open | USER (XAE) + REMOTE | H |
| MS-05 | Hydraulics telemetry over MQTT and Influx | REMOTE | L |
| MS-06 | Azimuth limit switch path into the brake (`bAzimutLimitSwitch`) | REMOTE (read) + USER (XAE) | M |
| MS-07 | Soft limits: command beyond limit per axis | REMOTE | M |
| MS-08 | Derotator out-of-range while tracking | REMOTE | M |
| MS-09 | Elevation and focus hardware limit inputs | USER (XAE) + REMOTE | H |
| MS-10 | Limit reached during slew / track, recovery | REMOTE | H |
| MS-11 | Auxiliary temperature sensors and wire-break guard (MONETS#17/#26) | REMOTE + USER (XAE) | L |
| MS-12 | Mode switching, E-Stop and readiness wiring in MAIN | USER (on site) + REMOTE | M |
| MS-13 | Error aggregation: `MasterError`, cabinet blink codes, reset paths | USER (XAE) + REMOTE | M |
| MS-14 | VISU bindings (MONETS#27 and Focus page) | REMOTE (read) + USER (XAE) | L |
| MS-15 | MQTT command interface: endpoint and verbs | REMOTE | M |
| MS-16 | Event log content (`MONETS/Log`) for project-level events | REMOTE | L |

## Per-test steps

### MS-01. Oil level reading and scaling (15 min) [REMOTE, L]

Reference: general plan T7 covers fill time and the 140 s watchdog; this test only checks the
MONETS scaling parameters.

1. ADS read `MAIN.HydraulicsControl.InOilLevel` and `MAIN.HydraulicsControl.fOilLevel` ten times
   over one minute with the hydraulics off.
2. Compute `(InOilLevel - 3500) / (10200 - 3500) * 100` and compare with `fOilLevel`.
3. Read `bOilPanMaximum`, `bSuctionRunning`, `srSuctionTrigger.Q1` (if exposed).

Expected: `fOilLevel` matches the formula to 0.1 %; values below 0 or above 100 are possible
because clipping is off. Suction is off while `fOilLevel < 50`.
Pass: formula matches, readings stable (spread < 2 %). Fail: mismatch, or suction running with
level < 50 % and no float switch. Abort: none (read only). Restore: nothing changed.

### MS-02. Oil level sensor broken / out of range (30 min) [USER (XAE) + REMOTE, M]

Purpose: a broken analog wire on the oil level sensor reads 0 (or a fixed value). With
`cut := FALSE` this gives `fOilLevel` around -52 %, which (a) is published to MQTT/Influx as a real
value and (b) keeps the suction trigger in reset, so suction only starts on the float switch
`IbOilpanMaximum`. There is no plausibility check comparable to the temperature wire-break guard.

1. Hydraulics off (`MAIN.HydraulicsControl.bEnable = FALSE`), brake closed.
2. USER (XAE): force `MAIN.HydraulicsControl.InOilLevel := 0`.
3. REMOTE: read `fOilLevel`, `bError`, `bOilWarning`; watch `MONETS/Telemetry` for
   `hydraulics/base/OilLevel` and `MONETS/Log` for any event for 60 s.
4. USER: force `InOilLevel := 16383` (above max): expect `fOilLevel` about 192 %, suction
   trigger set, `ObOilSuctionEnable = TRUE`. **Release immediately** after reading (within 5 s) if the
   suction pump actually starts with the hydraulics off; record whether it did.
5. USER: release all forces.

Expected (current code): step 3 shows about -52 %, no error, no warning, value published.
Step 4 sets the suction output; if `IbSuctionRunning` stays FALSE, `bSuctionFailure` after 15 s.
Pass: behaviour as described and documented. A fail here means the observed behaviour differs from
the code; in either case record as a finding that an invalid oil level is not flagged (proposed
issue: add a range guard like MONETS#17 and do not publish invalid levels).
Abort: suction pump running for > 10 s, any hydraulics alarm on the cabinet not listed above.
Restore: release forces, `MAIN.HydraulicsControl.bReset := TRUE` (ADS), check `bError = FALSE`.

### MS-03. Oil low / reserve / hot / cold and the telescope (45 min) [USER (XAE) + REMOTE, H]

Code facts: `bOilLow` (from `NOT IbOilLow`) sets the latched `bError`, which stops the pump
(`rsOilPumpState` reset); no pump means the brake closes (`NOT bPumpRunning`).
`bOilWarning` (reserve), `bOilHot`, `bOilCold` and `bOilFilterDirty` are warnings only: they do
not stop anything in the hydraulics FB; in MAIN `bOilWarning`, `bOilHot`, `bPressureFailure` give
the cabinet blink code `slow` (only if no safety or axis error is present).

1. Telescope powered (`command power=1`), at park, not tracking. Hydraulics running, brake open,
   `MAIN.HydraulicsControl.bBrakeOpen = TRUE`.
2. USER (XAE): force `IbOilReserve := TRUE`. REMOTE: read `bOilWarning`, cabinet blink state, MQTT
   `OilReserve`, `MONETS/Log`. Release force.
3. USER: force `IbOilHot := TRUE`, same reads (`bOilHot`, `OilHot`). Release. Repeat with
   `IbOilCold := FALSE` (oil cold is active low). Release.
4. Telescope stationary at park. USER: force `IbOilLow := FALSE` (oil low is active low).
   REMOTE: read `bOilLow`, `bError`, `ObOilPumpEnable`, `bPumpRunning`, `bBrakeOpen`,
   `MAIN.TelescopeControl.bError`, axis enables, `MasterError` on MQTT.
5. USER: release the force. REMOTE: `bError` must stay TRUE (latched). Clear via ADS
   `MAIN.HydraulicsControl.bReset := TRUE` or MQTT `command reset=1`; record which one clears it.
6. Restart hydraulics (`command power=1`) and check the brake reopens.

Expected: steps 2-3 warnings only, no motion change, blink `slow`. Step 4: `bError` within one
cycle, pump off, brake closed (after pump stops), `electronics/base/MasterError = true` within 5 s,
telescope control reports the brake/hydraulics problem and does not move. Step 5: error latched until
reset.
Pass: all as expected, telescope never moves with brake closed. Fail: pump keeps running with oil
low, brake stays open without pump, error not latched or not resettable.
Abort: any axis motion while the brake is closing; drive following error. Restore: forces released,
reset, hydraulics running or off as before, telescope parked.

### MS-04. Oil pressure loss with the brake open (20 min) [USER (XAE) + REMOTE, H]

Reference: general plan T6 covers sensor polarity. This checks the sequence, telescope stationary.

1. Telescope at park, powered, brake open.
2. USER: force `IbOilPressureOK := FALSE`.
3. REMOTE: time from force to `bBrakeOpen = FALSE` (expect about 2 s, `tonPressureLossDelay`), and
   to `bPressureFailure = TRUE` (30 s, then `bError`, pump off).
4. Release force, reset, power on again.

Pass: brake closes after 2 s, pressure failure after 30 s, error latched and resettable.
Abort: axis motion. Restore: release, `command reset=1`, `command park=1`.

### MS-05. Hydraulics telemetry over MQTT and Influx (15 min) [REMOTE, L]

Reference: general plan T3 covers cadence of the general telemetry.

1. Subscribe to `MONETS/Telemetry`. Toggle nothing; record all `hydraulics/base/*` fields
   (`OilLow`, `OilHigh`, `OilMaximum`, `OilReserve`, `OilPressureOK`, `OilFilterDirty`, `OilCold`,
   `OilHot`, `BrakeOpen`, `BrakeClearing`, `PumpRunning`, `SuctionRunning`, `OilLevel`,
   `StatusWord`) for 5 min.
2. Compare each value with the ADS value of the same symbol.
3. Check the same fields arrive in Influx with correct types (bool vs float).

Expected: fields are published when `StatusWord` changes (not periodically); `OilLevel` changes
alone may not trigger a publish (statusWord bit 15 tracks level changes - record the behaviour).
Pass: every field present at least once, values match ADS. Fail: missing field, wrong type, stale
value for > 5 min after a change. Restore: none.

### MS-06. Azimuth limit switch into the brake (20 min) [REMOTE read + USER (XAE), M]

Code fact: `FB_HydraulicsControl.bAzimutLimitSwitch` closes the brake, but MAIN does not pass it,
so it is always FALSE in the MONETS project.

1. REMOTE: read `MAIN.HydraulicsControl.bAzimutLimitSwitch` (expect FALSE).
2. USER: check in XAE whether an azimuth hardware limit switch input exists in the I/O tree and
   where it is linked (drive digital input, TwinSAFE, or nothing). Record it.
3. Decide (with the maintainers) whether the missing wiring is intended (limit handled by the drive
   or safety only) or a bug; open an issue if a bug.

Pass: the azimuth hardware limit path is identified and documented. No motion in this test.

### MS-07. Soft limits: command beyond the limit (40 min) [REMOTE, M, motion]

Code facts: azimuth and derotator clamp the target with `LIMIT(fMinPosition, fPosition,
fMaxPosition)`, elevation clamps to 0..90 in the library; focus to 0..100.

1. REMOTE: read `MAIN.AzimuthControl.fMinPosition/fMaxPosition` (-274/270),
   `MAIN.DerotatorControl.fMinPosition/fMaxPosition` (-90/360) to confirm the MAIN values are live.
2. Elevation: `command azimuth=175,elevation=95,slew=1`. Expect target clamped to 90 or the command
   rejected by telescope control; record which. Stop if elevation passes 88 without slowing.
   Then `command park=1`.
3. Elevation low: `command azimuth=175,elevation=-5,slew=1`; record clamp/rejection; the mount
   must not go below the telescope control's minimum elevation. Stop if elevation < 15 with covers
   open. Park.
4. Azimuth: slew to `azimuth=265`, then command `azimuth=280` (beyond +270). Expect clamp at 270
   or rejection; then to `azimuth=-270` and `azimuth=-280`. Record `MAIN.AzimuthControl.fPosition`
   vs requested. Park.
5. Focus: `command focus=120` and `command focus=-5`; expect clamp to 100 / 0, or rejection.
   Return to 36.0.
6. Derotator: write via the telescope commands only (no direct derotator command); see MS-08.

Pass: no axis target ever outside the limits in the table; MQTT log shows a warning on clamping or
rejection (record if silent). Abort: an axis moves beyond its limit value by more than 1 deg / 1 mm.
Restore: `command park=1`, check all axes at park positions.

### MS-08. Derotator out-of-range while tracking (30 min) [REMOTE, M, motion, roof closed]

Reference: general plan T4 (derotator range stop) - repeat only if T4 was not done on the build
under test. The project-specific part is the range -90..360 set in MAIN.

1. Track a target whose field rotation will reach a limit within 10 min (with the roof closed,
   tracking is motion only). Watch `MAIN.DerotatorControl.fActualPosition`.
2. At the limit expect `Enable_Positive`/`Enable_Negative` to block further motion and the
   derotator error flag (`ActualPosition > fMaxPosition` path) to be set.
3. `command stop=1`, `command reset=1`, `command park=1`.

Pass: derotator stops at -90 / 360 (within 1 deg), error visible, recovers after reset + park.

### MS-09. Elevation and focus hardware limit inputs (40 min) [USER (XAE) + REMOTE, H]

Elevation enable bits come from the drive digital inputs (`inDigitalInputs.0` positive,
`.1` negative; TRUE = free). Focus uses `IbLimitNear`/`IbLimitFar` (TRUE = at limit; far is also the
homing cam).

1. Telescope at home (el 40), stationary. USER: force the elevation positive-free bit FALSE (simulated
   upper limit switch) via the input copy in XAE.
2. REMOTE: `command azimuth=175,elevation=45,slew=1` - expect no upward motion and an error/warning;
   `elevation=35` - expect downward motion allowed. Stop at 35.
3. USER: release; repeat with the negative bit and commands to 30 / 40.
4. Focus at 36: USER forces `MAIN.FocusControl.IbLimitFar := TRUE`; REMOTE `command focus=40`:
   expect blocked; `focus=34`: allowed. Release. Same with `IbLimitNear` and `focus=34` / `38`.
5. Check the Focus VISU lamps for `IbLimitNear` / `IbLimitFar` follow the forces.

Pass: motion in the blocked direction never starts, the other direction works, errors resettable.
Abort: motion in the blocked direction. Restore: release all forces, `command reset=1`, park.
Real limit switch actuation (by hand) is on site only, see general plan T10.

### MS-10. Limit reached during slew/track, recovery (30 min) [REMOTE, H, motion]

1. Slew azimuth from park (175) to 268, then command a track whose azimuth path runs past 270.
2. Observe: telescope control stops or flips; `MAIN.TelescopeControl.bError`, `bTracking`,
   `bStopped`; MQTT log.
3. Recovery: `command stop=1`, `command reset=1`, `command park=1` (or `gohome=1`).

Pass: no axis passes the soft limit, a clear error/log message, recovery with reset + park only
(no XAE needed). Fail: axis passes the limit, or recovery requires XAE.
Abort: drive error, azimuth beyond 270.5. Restore: park.

### MS-11. Auxiliary temperature sensors and wire-break guard (30 min) [REMOTE + USER (XAE), L]

Code: `FB_MonetsTelescopeAuxiliary` reads `tempSensorM1`, `tempSensorM1cell`, `tempSensorM2`,
`tempSensorFlange` through MONETcommon `FB_MonetTempSensor`; `SendTelemetry` publishes
`telescope/dome/AUXILIARY.SENSOR[1..4].VALUE` only while `validM1`/`validM2`/`validM1cell`/
`validFlange` is TRUE (index 1 M1, 2 M2, 3 M1_CELL, 4 FLANGE). NAME and UNIT are always published.

1. REMOTE: read `MAIN.TelescopeAuxiliary.tempM1..tempFlange` and `valid*`; compare with MQTT and
   Influx over 1 min (5 s period).
2. USER: force `MAIN.TelescopeAuxiliary.tempSensorM2` to the raw value of a broken wire (the value
   that maps to -50 C; check the `FB_MonetTempSensor` scaling, typically -500 for 0.1 C/LSB).
3. REMOTE: `validM2` FALSE, `AUXILIARY.SENSOR[2].VALUE` no longer published, no -50 in Influx,
   NAME/UNIT still published; check `MONETS/Log` for a sensor event naming `TEMP_M2`.
4. USER: force a plausible value (e.g. 15 C), check `validM2` returns TRUE and VALUE resumes.
5. Release.

Pass: no invalid value reaches MQTT/Influx, flag recovers. Restore: release forces.

### MS-12. Mode switching, E-Stop and readiness wiring (45 min) [USER on site + REMOTE, M]

Reference: E-Stop and safety reset in general plan T8/T11; here only the MAIN wiring.

1. REMOTE: read `MAIN.eTelescopeMode`, `MAIN.ready`, `MAIN.CabinetControl.KeyLocal`.
2. USER (on site): key to local. Expect `eTelescopeMode = off` (hydraulics off) or `manual`
   (hydraulics on), `TelescopeControl.SuspendAutomatic()` running: MQTT commands must have no effect
   (try `command park=1` while parked and `command focus=37`; record).
3. USER: start/stop buttons toggle `HydraulicsControl.Enable` only in local; brake clearing follows
   `CabinetControl.AreBreakesCleared()`.
4. Key back to remote: `eTelescopeMode = automatic`, startup readiness re-established.
5. `MAIN.ready` (= `PowerMonitoring.phaseOK`) is passed as `bMainReady` and `SafetyHandling.estop` as
   `bEstopTriggered` to telescope control: with an E-Stop pressed (USER, telescope parked) check
   `TelescopeControl` refuses `power`/`slew`, and `electronics/base/MainReady` is published every 5 s.

Pass: commands ignored in local, accepted in remote, E-Stop blocks motion. Restore: E-Stop released,
safety reset, key remote, park.

### MS-13. Error aggregation and reset paths (30 min) [USER (XAE) + REMOTE, M]

Code: `MAIN.error := CoverControl.Error OR HydraulicsControl.Error` (focus error is commented out;
axis errors are not included). Blink priority: safety error `short` > axis error `long` > oil
warning/hot/pressure `slow` > `error` `on` > off. Reset button (cabinet or pendant) resets covers,
hydraulics and, in automatic only, telescope control.

1. Create a hydraulics error (as MS-03 step 4, telescope parked) and confirm `MasterError = true`.
2. While it is active, create an oil warning (`IbOilReserve`): blink stays per priority (record).
3. Check that a focus error or an axis error alone gives `MasterError = false` (expected by code;
   record as a finding if this is not desired).
4. Reset via MQTT `command reset=1`, then via ADS `MAIN.HydraulicsControl.bReset`; USER on site:
   cabinet reset button. Record which paths clear which errors.

Pass: behaviour matches the code description and is documented. Restore: release, reset, park.

### MS-14. VISU bindings (20 min) [REMOTE read + USER (XAE), L]

1. MONETS#27: the Cover VISU binds `MAIN.CoverControl.bWarning`, which does not exist in
   `FB_MonetCoverControl`. USER: open the Cover page, check the lamp state and the compile messages.
2. Focus VISU binds `MAIN.AzimuthControl.bCalibrated`, `MAIN.AzimuthControl.fMinPosition` and
   `fMaxPosition` alongside the focus symbols - likely a copy-paste error (shows -274/270 on the
   focus page). Check and open an issue if confirmed.
3. For each page (Azimuth, Elevation, Derotator, Focus, Hydraulics, Cover, Safety, Telescope),
   compare 3 displayed values with ADS reads.
4. Roof page: view only. Do not press any button.

Pass: all bound values match ADS except the documented issues. No motion.

### MS-15. MQTT command interface (30 min) [REMOTE, M]

Reference: general plan T12 covers pyobs and a normal night. Here only the MONETS endpoint.

1. Confirm subscription on `MONETS/Telescope/SET`, telemetry on `MONETS/Telemetry`, log on
   `MONETS/Log` (MAIN `fbComm` parameters).
2. Send each verb once with the telescope in a safe state: `command power=1`, `gohome=1`, `park=1`,
   `focus=36.5`, `slew` to az 180 / el 50, `track` (roof closed, motion only), `stop=1`, `reset=1`.
3. Send malformed lines (`command azimuth=abc,slew=1`, unknown verb, empty payload): expect
   no motion and a log message.
4. MQTT watchdog: covered in general plan; do **not** test here because its action includes closing
   the roof (it would only close an already closed roof, but the test is left to the general plan).

Pass: each verb has the documented effect, malformed input ignored. Restore: park.

### MS-16. Event log for project-level events (15 min) [REMOTE, L]

During MS-02 to MS-13 record all `MONETS/Log` entries. Check that each forced fault produced
exactly one entry on set and one on clear, with a readable text. Pass: no fault without log entry.

## Results

| ID | Date | Build / commit | Executed by (role) | Status (pass/fail/skip) | Notes / issue |
|---|---|---|---|---|---|
| MS-01 | | | | | |
| MS-02 | | | | | |
| MS-03 | | | | | |
| MS-04 | | | | | |
| MS-05 | | | | | |
| MS-06 | | | | | |
| MS-07 | | | | | |
| MS-08 | | | | | |
| MS-09 | | | | | |
| MS-10 | | | | | |
| MS-11 | | | | | |
| MS-12 | | | | | |
| MS-13 | | | | | |
| MS-14 | | | | | |
| MS-15 | | | | | |
| MS-16 | | | | | |

## Left open / not testable remotely

- Anything with the roof open: tracking under sky, MQTT watchdog park + roof close sequence, roof
  limits (`max_position_1 := 205`, `max_position_2 := 203`, `max_position_diff := 3`,
  `limit_slowdown := 5`). Out of scope.
- Real actuation of limit switches, E-Stop, key switch, cabinet buttons and pendant: on site only.
- Real low oil level or draining oil: not done; only forced inputs.
- Covers: no cover soft limits in MAIN; cover travel and errors are in the general plan (T0, T15).
- Weather check (`FB_WeatherCheck`) is commented out in MAIN: not tested.
- Wake&Shake (`MAIN.WakeAndShake`, marked untested in MAIN) and `FB_SoECommand`: USER (XAE) only,
  separate procedure.
- Elevation soft limits 0..90 are hard-coded in the library, not configurable in MONETS.
- Minimum elevation enforced by telescope control (e.g. for park/covers) is library logic.
