# Wake&Shake on MONET/S: findings (2026-10-28 … 2026-10-30)

Commissioning notes on the AX5000 commutation search (Wake&Shake, P-0-0160) for the
elevation (AX5125-0000-0214) and azimuth drives. PLC: MONETSRuntime 1.6.2 with BROTLib
0.16.0, HalfBROT 0.10.2, MONETcommon 0.11.2, MONET Roof 1.1.9, AstroBROT 0.5.0.

Statements marked *(inference)* are interpretations of the data, not confirmed by Beckhoff.

## Summary

- **Manual route works:** Wake&Shake in the Drive Manager, then `reset=1` + `power=1` → both
  axes home normally. On elevation, P-0-0160 usually fails with F616 unless P-0-0166 is run
  first (sometimes 0166 twice).
- **Automatic route (P-0-0165 Activation = "On enable request") is unreliable on elevation:**
  1 clean success, several F616 failures on cold starts. Azimuth looked fine when it got its
  turn.
- **An activation in XAE counts as a drive restart** for P-0-0165: the next enable runs the
  Wake&Shake. A reset + new `power=1` without a restart does *not* run it again.
- **P-0-0166 readings scatter strongly** on elevation (‑30° … ‑155° electrical), even right
  after a successful 0160. They are not usable to judge the commutation right now.
- **Beckhoff support:** no SoE password needed, because Wake&Shake can run "on enable"
  (P-0-0165). The PLC-side SoE approach (branch `feature/wake-and-shake-develop`, worktree
  `MONETS-ws`) is therefore not needed.

## Background

- P-0-0160: commutation search (Wake&Shake). Finds the rotor's electrical angle; afterwards
  the commutation is valid until the next drive restart.
- P-0-0165: Wake&Shake configuration (mode, activation, currents, times, Kp, …).
  Activation "On enable request" = runs at the first enable after a drive restart.
  Activation "Manual" = an enable without valid commutation just errors (F107 seen), safe.
- P-0-0166 / P-0-0167: motor and feedback connection check (rotating vector, 50 %,
  90°/pole pairs, 30°/(pp·s)). Reports *EqualDirections* and *Commutation position
  difference (electrical)*; expected ≈ 0° (350…10°). It is a check, not a calibration.
- P-0-0057: electrical commutation offset.
- Writing P-0-0160/0166 over SoE from the PLC is password-protected (SoE 0x7009) until
  the Drive Manager has unlocked the drive since its last power-up.
- PLC window for a Wake&Shake: only at the first power-on after a PLC restart with an
  axis error is there no emergency park. `FB_ElevationControl` sets bReady =
  brake open + `tBrakeSettle` (default 3 s) + hard-coded 6 s. A drive Wake&Shake longer
  than that would let homing start too early (HalfBROT change needed if that becomes a
  problem).

## Test log

| When | Setup | Result |
|---|---|---|
| 10-28 | P-0-0165 switched Manual → On enable request | `power=1`: elevation F616, W&S started 0.5 s after brake open → HalfBROT#130 (settle timer) |
| 10-29 15:27 | automatic, cold | **Success on both axes** (el +2° at 50 % torque, az ±1.6°, max lag 0.4/0.8°) – `WakeAndShake.svdx` |
| 10-29 | automatic, cold, 2× | Elevation F616 (+F601, F10B): axis moved only ~0.01–0.03°, drive gave up after 0.4 s |
| 10-29 | `tBrakeSettle` 8 s instead of 3 s | Still F616 → brake timing is not the cause |
| 10-30 | elevation W&S waiting time 150 → 1500 ms | Still F616 → waiting time not the cause |
| 10-30 | manual W&S | El 0160 failed 2×, worked after 0166. Homing OK afterwards |
| 10-30 | power-ups with F123 | F123 (= P-0-0166 failed) 18.01 s after each brake opening. PLC 1.6.2 contains no SoE/0166 code → trigger unexplained |
| 10-30 | automatic, after activation | Clean: az activity brake+4.2 s (≤70 %, ±0.6°), el pulse brake+8.1 s (35 %, 0.073°), then homing, el peak 98 % at homing start. *(inference: this was a successful W&S)* |
| 10-30 11:54 | automatic, after activation | Elevation F616 4.8 s after brake open, 0.12° motion, 41 % torque. Azimuth had not started |
| 10-30 | reset + `power=1` (no restart) | No W&S motion on elevation. **Azimuth NC 0x4260** during homing at 2°/s, torque spike ‑99.8 % (meaning not looked up, unexplained) |
| 10-30 | Activation = Manual, after activation | Elevation **F107** 3 s after brake open, no motion (meaning not looked up; expected "no commutation") |
| 10-30 | manual W&S (see 0166 table) + reset + `power=1` | **Homing OK**, ready after 235 s at el 40° / az 175°, el peak torque 45.7 %, max 3.0°/s |

## P-0-0166 results (10-30, manual)

| Axis | P-0-0057 | Before/after successful 0160 | EqualDirections | Difference (electrical) |
|---|---|---|---|---|
| Az | 270° | after W&S | **No** | 146.58° |
| El | 270° | before (0160 had failed) | Yes | 323.69° (‑36°) |
| El | 270° | before (0160 failed again) | Yes | 289.97° (‑70°) |
| El | 270° | **after** | Yes | 329.84° (‑30°) |
| El | 240° | after | Yes | 204.74° (‑155°) |

- Changing P-0-0057 by 30° moved the result by ~125° → the check is not reproducible
  enough on elevation; the 240° idea (startup-list value) did not help. Offset is back at 270°.
- Azimuth homed and ran normally despite 146.58° / "No".

## P-0-0165 on the elevation drive (export 2026-10-30 12:10)

Actual drive values, compared with the repo's `Elevation Startup List.csv` and
`Azimuth Startup List.csv`. The startup lists hold P-0-0165 only as raw bytes; the
decoding below assumes 16-bit words after a 2-word length header (0x3C = 60 bytes) and is
not verified. Units: % and ° in 0.1 steps.

| Field | El drive (actual) | Default | El startup list | Az startup list |
|---|---|---|---|---|
| Command mode | Wake and Shake (1) | Static current vector (0) | 1 | 1 |
| Activation | **Manual (0)** | Manual | 1 (= on enable?) | 1 |
| Max. current deviation | 10.0 % | 10.0 % | 10.0 % | 10.0 % |
| Static current vector: torque | 50.0 % | 50.0 % | 50.0 % | 50.0 % |
| Static current vector: duration | 3000 ms | 3000 ms | 10000 ms | 3000 ms |
| W&S first phase torque | **50.0 %** | 50.0 % | 50.0 % | **60.0 %** |
| W&S first phase ramp up | 100 ms | 100 ms | raw 1000 (?) | raw 1000 (?) |
| W&S second phase torque | 50.0 % | 50.0 % | 50.0 % | 60.0 % |
| W&S second phase ramp up | 500 ms | 500 ms | raw 1000 (?) | raw 1000 (?) |
| Commutation pos control Kp | 0.04 | 0.04 | raw 100 (?) | raw 100 (?) |
| First phase pos monitoring limit | 0.5° | 0.5° | 0.5° | 0.5° |
| First phase step width | 22.5° | 22.5° | 22.5° | 22.5° |
| First phase waiting time after step | **1500 ms** | 150 ms | 500 ms | 500 ms |
| Second phase duration | 3000 ms | 3000 ms | 7000 ms | 6000 ms |
| Error monitoring (range of motion) | 90.0° | 90.0° | 45.0° | 90.0° |

- The elevation drive runs on defaults except mode and waiting time.
- The repo startup lists differ from the drive in several fields. No P-0-0165 init command
  was found in `MONETS.tsproj` / `.xti` files, so the lists look like exports and probably
  aren't applied on activation. Not verified.
- Fields marked (?) don't match the drive under the assumed units, so the decoding of
  those words is uncertain.

## Hypotheses (unconfirmed)

1. *F616 = too little motion in the first search step.* The axis then moved < ~0.03°.
   0166 helps because it physically moves the rotor (50 % current, large distance) and
   breaks stiction or leaves it in a better position. **Not proven**, no measurement of
   the rotor motion during 0166.
2. *Wake&Shake current too low on elevation.* The drive runs both phases at the default
   50 % (see P-0-0165 table). The repo's `Azimuth Startup List.csv` has 60 % for azimuth
   (decoded from raw bytes, see below), and azimuth has not failed so far.
3. *F616 comes from the first-phase position monitoring.* "First phase pos monitoring
   limit" is 0.5°; in the failing runs the axis barely moved. Whether F616 is tied to this
   limit or to "Error monitoring (range of motion)" (90°) is not confirmed.
4. Other levers: Kp (commutation pos control) = 0 (Beckhoff), "Static current vector"
   mode, 1–3 s waiting time, balanced axis, same start position, 40–60 % current, repeat
   3× and verify the angle (Potsdam colleague).

## Open questions

- Why does 0160 fail on elevation unless 0166 runs first? (→ Beckhoff)
- Why do 0166 results scatter by ≥ 35° and stay ~30° off after a successful 0160?
  Should a good 0160 give ≈ 0°? (→ Beckhoff)
- Azimuth 0166 "EqualDirections = No" with 146.58°: wiring/direction, or meaningless?
- Meaning of NC 0x4260 (azimuth, homing) and drive F107 (elevation, Manual mode).
- What triggered F123 (P-0-0166 failed) 18 s after brake opening?
- Are the repo startup lists applied anywhere, or just stale exports?

## Current settings (end of 2026-10-30)

| Item | Value | Note |
|---|---|---|
| P-0-0165 Activation | **Manual** (both axes) | Startup lists unchanged |
| El W&S waiting time | 1500 ms | default 150 ms, changed for test |
| P-0-0057 elevation | 270° | Startup list says 240°, azimuth list 280° |
| P-0-0167 current elevation | 50 % | Startup list says 20 % |
| NC lag / position range limits | changed for tests | originally 5.0° on elevation; restore |
| `tBrakeSettle` | 3 s (default) | 8 s test reverted, activation downloaded the revert |

## Next steps

1. Export P-0-0165 for azimuth too, to confirm its 60 % (elevation is done, see table).
2. Raise the elevation Wake&Shake first/second phase torque from 50 % step by step
   (e.g. 60 %, 75 %), cold start each time,
   record success / F616 and the motion. Change one parameter at a time; then try Kp = 0 or
   Static current vector.
3. Ask Beckhoff about 0160 vs 0166 and the scattering 0166 results (data above).
4. Once a setting holds over several cold starts: write P-0-0165 into both startup lists.
5. Restore NC lag / range limits and decide on the waiting time.

## Data and tools

- Scope project `MotorTuning/WakeAndShake.tcscopex` (29 channels, 180 s); recordings
  `WakeAndShake.svdx`, `_1`, `_2`. Decode with `IAG/tools/decode_svdx.py <file.svdx> [out.csv]`.
- MQTT on `MONETS/Telescope/SET`: `command power=1`, `reset=1`, `park=1`.
- ADS (PLC runtime, port 851): `MAIN.ElevationControl.*`, `MAIN.AzimuthControl.*`,
  `MAIN.HydraulicsControl.*`, `MAIN.TelescopeControl.fReadyState` (‑1 error, 0 off,
  0.8 powering/homing, 1 ready).
