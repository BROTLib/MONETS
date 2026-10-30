# Wake&Shake on MONET/S: findings (2026-10-28 … 2026-10-30)

Commissioning notes on the AX5000 commutation search (Wake&Shake, P-0-0160) for the
elevation (AX5125-0000-0214) and azimuth drives. PLC: MONETSRuntime 1.6.2 with BROTLib
0.16.0, HalfBROT 0.10.2, MONETcommon 0.11.2, MONET Roof 1.1.9, AstroBROT 0.5.0.

Statements marked *(inference)* are interpretations of the data, not confirmed by Beckhoff.

## Summary

- **Result (end of 2026-10-30): confirmed.** After the change of `elevationPark` to 40°,
  three consecutive power-ups (one from 40° with open covers, two from fully parked with an
  activation in between) all passed with the same W&S swing (+4.0°) and a lag of ~0.11°.
  Not yet tested: a real power cycle of the drives (only XAE activations), several
  consecutive days, the winter/cold case, azimuth with Kp 0.
- **Update 2026-10-30 afternoon: first fully successful automatic cold start** with
  elevation P-0-0165 at 60 % (both phases), 500 ms waiting time, **Kp = 0**, start near the
  balanced elevation (~30–35°). Kp 0.04 made the second phase swing 6–8° and left a wrong
  commutation (0166: 162.56°, axis stalled at 63 % torque). One run only: repeat before
  making it permanent.

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
| 10-30 12:19 | automatic, **60 %** torque, 1500 ms, from park (el ~60°) | **No F616.** First phase 8 s of small steps, second phase 3 s at exactly 60 % moved the axis **+6.3°**, then ~2 s with 0 reported torque while it dropped 2.5°. PLC `bReady` came at brake + 9 s, during the second phase, so homing started too early → following error 0x4550 (5.011° > 5.0°) |
| 10-30 12:23 | automatic, 60 %, **500 ms**, from where the last error left the axis | **F616** (+F601, F10B) 3.8 s after brake open; the axis sagged steadily 0.40° downwards with no reported torque. Start position differed from the 1500 ms run |
| 10-30 12:41 | automatic, 60 %, 500 ms, from **el 35°** (balanced) | **No F616.** First phase 5 s with tiny motion (≤ 0.08°); second phase 3.3 s at 60 % swung the axis **−8.2°** (up to 4.4°/s). W&S ended at brake + 8.6 s, `bReady` at brake + 9 s; the NC set position was still at the start → following error 8.28° > 5.0° (0x4550) right away. Azimuth W&S had just started |
| 10-30 12:45 | automatic, 60 %, 500 ms, from ~27° (estimated), NC lag limit and range window **15°** | No F616; second phase swung +6.7°. After `bReady` the NC pulled back at −100 %, then from brake + 14 s the axis **stood still while the drive held a constant ~63 % torque**, lag growing at ~1°/s until 15.02° > 15° (0x4550). Drive warning **ECD2** at exactly that moment. Azimuth 0x4260 "lost controller enable while executing" = consequence of the elevation error (PLC disabled all axes during azimuth homing). The 60 % / 1500 ms run showed the same 63 % plateau |
| 10-30 12:5x | automatic, 60 %, 500 ms, **Kp = 0**, from ~31–33° (estimated), NC limits 15° | **Full success.** First phase no motion, second phase swing only −2.5° (returned to −0.3° by itself). NC took over with max lag 0.016°, homing torque 4–16 % (peak 71 % at start). Both axes homed, ready after 240 s at el 40° / az 175° |
| 10-30 13:0x | same, from **40°**, NC limits back at **5°** | **Success.** Second phase swing +4.05°, back to +0.9°. Max lag after takeover 0.071°, homing torque peak 82 % at start. Ready after 246 s at el 40° / az 175° |
| 10-30 13:1x | same, from **60°** (park elevation), NC limits 5° | **Passed, but marginal.** First phase sagged −0.13°, second phase swing +6.1°, axis still moving at `bReady`. Takeover at ±100 % torque, max lag **3.2°** (limit 5°), brief 63 % plateau; > 55 % torque in 360 samples spread over the whole homing (the two balanced runs: 3 samples each). Ready after 244 s. Suggests a worse commutation from the unbalanced start *(inference)* → park lower (`elevationPark` 60 → 40 = `elevationHome`, MAIN.TcPOU) |
| 10-30 afternoon | **Repeat cycles** with Kp 0 / 60 % / 500 ms, NC limits 5°, `elevationPark` 40°: rep1 from 40° with open covers; rep2 and rep3 from fully parked (covers closed, `park=1` → activation → `power=1`) | **3 of 3 OK.** W&S swing −0.04…+4.10° / −0.02…+4.04° / −0.02…+4.04°, position at `bReady` ≈ 1.8°, max lag after takeover 0.115 / 0.108 / 0.119°, > 55 % torque in 4 / 4 / 6 samples, ready after 250 / 289 / 292 s (covers open first from parked: enable at ~43 s, brake +5 s, `bReady` brake + 9 s). `park=1` ends at 40.01° with covers closed in ~40 s |
| 10-30 | manual W&S at 60 % | PLC warning "Elevation torque is high" (|torque| > 50 % for > 1 s, harmless); elevation moved ~10° during the manual W&S |

## P-0-0166 results (10-30, manual)

| Axis | P-0-0057 | Before/after successful 0160 | EqualDirections | Difference (electrical) |
|---|---|---|---|---|
| Az | 270° | after W&S | **No** | 146.58° |
| El | 270° | before (0160 had failed) | Yes | 323.69° (‑36°) |
| El | 270° | before (0160 failed again) | Yes | 289.97° (‑70°) |
| El | 270° | **after** | Yes | 329.84° (‑30°) |
| El | 240° | after | Yes | 204.74° (‑155°) |
| El | 270° | after the **automatic** W&S at 60 % (12:45 run, axis stuck at 63 % torque) | Yes | **162.56°** |

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

## Elevation balance (2026-10-30, covers open, az 175°)

Holding torque (`ActTorque`, % ) at standstill, 5 s after each slew, averaged over 3 s
(20 Hz, std ≤ 0.2 %). Positions approached from below (up) and from above (down).
Imbalance = (up + down) / 2, friction = (up − down) / 2. Positive torque = towards zenith.

| Elevation | Up | Down | Imbalance | Friction |
|---|---|---|---|---|
| 25° | +4.45 | −8.21 | −1.9 | 6.3 |
| 35° | +7.41 | −4.67 | +1.4 | 6.0 |
| 45° | +8.03 | −3.57 | +2.2 | 5.8 |
| 55° | +8.47 | −3.16 | +2.7 | 5.8 |
| 65° | +10.99 | −0.56 | +5.2 | 5.8 |
| 75° | +12.10 | −0.11 | +6.0 | 6.1 |
| 85° | +12.37 | – | – | – |

- Friction is about 6 % and nearly constant.
- The axis is balanced at about 30–35°. Above that, gravity pulls it towards the horizon,
  rising to about +6 % at 75°; below it, slightly towards the zenith.
- At the park elevation (~60°) the imbalance is about +4 %, so with the brake open the
  axis tends to sag downwards, as seen in the failed 60 % / 500 ms run *(inference)*.
  Static friction is larger, but the Wake&Shake steps may break it loose.

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
4. *Imbalance disturbs the search.* See the balance table: start the Wake&Shake near
   30–35°, where the axis is balanced, instead of the park elevation (~60°).
5. Other levers: Kp (commutation pos control) = 0 (Beckhoff), "Static current vector"
   mode, 1–3 s waiting time, balanced axis, same start position, 40–60 % current, repeat
   3× and verify the angle (Potsdam colleague).

## Open questions

- Why does 0160 fail on elevation unless 0166 runs first? (→ Beckhoff)
- Why do 0166 results scatter by ≥ 35° and stay ~30° off after a successful 0160?
  Should a good 0160 give ≈ 0°? (→ Beckhoff)
- Azimuth 0166 "EqualDirections = No" with 146.58°: wiring/direction, or meaningless?
- Meaning of drive F107 (elevation, Manual mode). NC 0x4260 on azimuth is explained: "lost
  controller enable while executing" = knock-on of an elevation error during azimuth homing.
- Meaning of elevation drive warning **ECD2** (12:45 run, exactly when the axis stalled at
  63 % torque with Kp 0.04).
- Why did the azimuth run normally with a 0166 result of 146.58°?
- What triggered F123 (P-0-0166 failed) 18 s after brake opening?
- Are the repo startup lists applied anywhere, or just stale exports?

## Current settings (end of 2026-10-30)

| Item | Value | Note |
|---|---|---|
| P-0-0165 Activation | **On enable request** (elevation); azimuth unchanged | Startup lists unchanged |
| El W&S first/second phase torque | 60 % | default 50 % |
| El W&S Kp | 0 | default 0.04 |
| El W&S waiting time | 500 ms | default 150 ms |
| El NC max position lag / position range window | 5.0° / 5.0° | raised to 15° for two runs, restored |
| `elevationPark` (MAIN.TcPOU) | 40° (= `elevationHome`) | was 60°; commit 6cb8c1f, active after the next activation |
| P-0-0057 elevation | 270° | Startup list says 240°, azimuth list 280° |
| P-0-0167 current elevation | 50 % | Startup list says 20 % |
| `tBrakeSettle` | 3 s (default) | 8 s test reverted, activation downloaded the revert |

## Next steps

1. **Repeat the realistic cycle several times:** `park=1` (now at 40°, covers closed) →
   activation (drive restart) → `power=1` → check W&S swing, takeover lag, homing torque.
   Also check that parking at 40° closes the covers correctly.
2. Once it holds: write the elevation P-0-0165 values (60 %, 500 ms, Kp 0, On enable
   request) into `Elevation Startup List.csv`.
3. Azimuth: export P-0-0165 and consider Kp = 0 there too (no failures so far).
4. Look up ECD2; ask Beckhoff about Kp 0.04 vs 0 (wrong commutation, large second-phase
   swing), 0160 vs 0166, and the scattering 0166 results.
5. Long term: balance the elevation axis better (imbalance up to 6 % at 75°).
6. PLC: `bReady` is still brake + 3 s + 6 s fixed; the W&S now ends at about brake + 8.6 s,
   so there is little margin. Consider waiting for the drive (HalfBROT).

Superseded steps (done): raise torque to 60 %, try Kp = 0, restore NC limits.

## Data and tools

- Scope project `MotorTuning/WakeAndShake.tcscopex` (29 channels, 180 s); recordings
  `WakeAndShake.svdx`, `_1`, `_2`. Decode with `IAG/tools/decode_svdx.py <file.svdx> [out.csv]`.
- MQTT on `MONETS/Telescope/SET`: `command power=1`, `reset=1`, `park=1`.
- ADS (PLC runtime, port 851): `MAIN.ElevationControl.*`, `MAIN.AzimuthControl.*`,
  `MAIN.HydraulicsControl.*`, `MAIN.TelescopeControl.fReadyState` (‑1 error, 0 off,
  0.8 powering/homing, 1 ready).
