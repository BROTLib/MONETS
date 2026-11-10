# Motor tuning: the three sources disagree (2026-11-10)

Why this exists: after activations (MONETS updates) the drive parameters, up to "the whole motor tuning", were found lost
(P-0-0165 at factory defaults on 2026-11-09 17:11 and 2026-11-10). This note puts the values of the three places that hold a
tuning next to each other. It changes no drive and no project file; nothing here is verified on a running activation.

Sources:
- **Drive** = the Drive Manager export of 2026-11-09 17:16 (`DriveManager/parameter-lists-2026-11-09/`, after the MONETS update of that day).
- **tsproj** = the SoE init commands (all `Transition PS`) in `MONETS/MONETS.tsproj` (develop c6e4040; the local XAE-rebuilt tsproj has the same ones).
- **Startup list** = `Elevation Startup List.csv` / `Azimuth Startup List.csv` in the repo (Drive Manager startup lists, not applied by the project as far as found).

Scales: S-0-0100 raw 0.001, S-0-0101 raw 0.1 ms, S-0-0106 raw 0.1 V/A, S-0-0104 raw as the drive shows it.

| Parameter | Drive el | Drive az | tsproj el | tsproj az | tsproj derotator | Startup list el | Startup list az |
|---|---|---|---|---|---|---|---|
| S-0-0100 velocity loop gain | 20000.000 | 120000.000 | 70.311 | 299.995 | 0.097 | 12000.000 | 25000.000 |
| S-0-0101 velocity integral time | 10.0 ms | 30.0 ms | 8.0 ms | 8.0 ms | 8.0 ms | 10.0 ms | 50.0 ms |
| S-0-0104 position loop Kv | 0.08 | 0.08 | none | none | none | raw 7 | raw 6 |
| S-0-0106 current loop gain | 20.2 V/A | 40.5 V/A | 20.2 | 40.5 | 215.7 | 14.4 | 27.3 |
| S-0-0107 current integral time | 0.8 ms | 0.8 ms | 0.8 | 0.8 | 0.8 | 0.8 | 0.8 |

What follows from it:
- Current loop gain and integral time: drive and tsproj agree (the current loop was written from the motor data once and survives).
- **Velocity loop gain and integral time: none of the three sources agrees with the drive.** The drive holds 20000 / 120000, the tsproj
  70.3 / 300.0 (a factor of 285 and 400 lower), the startup lists 12000 / 25000.
- Because the tsproj commands run at every `PS` transition (each EtherCAT start, so each activation), they would write 70.3 / 300.0 into
  the drives again. The drive showed 20000 / 120000 at 17:16, so either they were not applied on that start, or the values were entered
  again after the activation (README of the export: P-0-0165 was set again before the export). **Which one is not known.**
- P-0-0165 (Wake&Shake) and the loop gains are in no project file; P-0-0165 has no init command at all.

Experiment that decides it (needs the user at the Drive Manager, nothing on the telescope moves):
1. Export the parameter lists of elevation, azimuth and derotator (the derotator was never exported) right now and keep them.
2. Activate the project again without changing any tuning; before the first power-on export the lists again.
3. Diff the two exports. Any difference in S-0-0100 / S-0-0101 / S-0-0104 / P-0-0165 is an activation overwrite; none means the
   earlier losses came from somewhere else (a drive restart, RAM-only writes, a replaced drive configuration).

If the activation does overwrite: the tuned values go into the tsproj as init commands (S-0-0100, S-0-0101, S-0-0104 and P-0-0165, IDN 32933, as raw bytes
from the export), or the old tsproj velocity commands are removed. That is a change to the project and needs the intended values
from the user first, they are not derivable from the repo.

## Result of the comparison, 2026-11-10 17:04 (exports in the user's Downloads, after the activations of that day)

Exports of elevation, azimuth and derotator at 17:04, diffed against the 2026-11-09 17:16 exports (parameters `S-0-*` and `P-0-*`, actual values):
- **Elevation and azimuth: no tuning parameter differs.** S-0-0100 (20000 / 120000), S-0-0101 (10 / 30 ms), S-0-0104 (0.08), S-0-0106, S-0-0107 and
  every other configuration parameter are identical. Only live values differ (positions, temperatures, loop times, mains voltage, status words, the
  power stage operating time, the communication error counters 185 to 189 on elevation and 180 to 184 on azimuth). So the activations of 2026-11-10 did **not** change the
  velocity loop of these two drives, although the tsproj holds other values for them: those init commands do not end up in the drive (or something
  writes the tuned values after them; not found).
- **Derotator (first export): the drive holds exactly the tsproj values** (S-0-0100 0.097, S-0-0101 8.0 ms, S-0-0106 215.7 V/A, S-0-0107 0.8 ms,
  S-0-0104 1.00), i.e. untuned defaults for the velocity loop (S-0-0100 default 0.100, Tn default 5.0 ms). It has no tuning of its own in the drive.
- **P-0-0165 (Wake&Shake) cannot be compared:** the parameter list CSV has the row empty (the parameter is not readable as a value in the list). It must be read in the
  Drive Manager dialog of the drive; the loss of 2026-11-09 17:11 is therefore not checked by this diff.

Conclusion for now: the activation by itself did not reset the elevation/azimuth loop tuning on 2026-11-10. What was lost earlier (and on
2026-11-09 17:11 P-0-0165) is not explained by this; the exports of 2026-11-09 17:16 and 2026-11-10 17:04 are the reference from which a loss can be detected from now on.
Next check: read P-0-0165 in the Drive Manager before and after the next activation, and keep an export per activation day.

## P-0-0165 on the elevation drive, 2026-11-10 17:09 (separate export of the parameter, Downloads)

Read after the activations of 2026-11-10 and before the next power-on: **all Wake&Shake settings are intact** and equal to the values set on 2026-11-09:
command mode Wake and Shake (1), activation **On enable request (1)**, first and second phase torque 60.0 %, ramp up 100 ms / 500 ms, commutation position control
Kp **0.00**, first phase waiting time after step **500 ms**, step width 22.5 deg, position monitoring limit 0.5 deg, second phase duration 3000 ms,
error monitoring 90.0 deg, static current vector 50 % / 3000 ms. Set value and actual value are the same.
So the activations of 2026-11-10 did not reset P-0-0165 either (on 2026-11-09 at 17:11 it was found at the factory defaults after the update of that day).
Still only in the drive: no project file holds P-0-0165, so a drive replacement or a reset to defaults would lose it.

## Correction and resolution, 2026-11-10 evening (read this before the sections above)

Where the XAE Startup list really lives: in `MONETS/MONETS.tsproj` as `<MBoxUserCmdData>` hex blobs (385 of them; layout `... 03 40 <IDN LE16> 00000000 <data> <comment>`;
elevation group around line 8584-8619, azimuth group around line 7119-7156). The `<Calc><SoE><InitCmd>` blocks quoted above are secondary copies; the S-0-0100/0101
values in them were aligned with the drive in #102, but the blobs are what the Startup tab shows. The statement above that P-0-0165 is "in no project file" was wrong:
the blobs held P-0-0165 with **factory defaults**, and it is written at every `PS` transition. So each activation of a project built from develop reset the Wake&Shake
parameters (the loss of 2026-11-09 17:11); the local XAE project had the tuned values, which is why 2026-11-10 activations from it kept them.

Fix on develop (direct commits, no PR): 0fa8790 sets the elevation P-0-0165 blob to the tuned bytes (Wake and Shake, activation On enable request, 60 % / 60 %, ramps 100 / 500 ms,
Kp 0.00, waiting 500 ms); c6edef5 sets the azimuth blob's activation to On enable request (other fields unchanged: 50 %, Kp 0.04, 150 ms). After c6edef5 all 385 blobs equal the
local XAE tsproj. Verified on 2026-11-10 17:53 after an activation of this project: parameter lists unchanged against 17:04 (el, az, derotator), P-0-0165 read in the Drive Manager
as intended, power-on / slew / gohome / park without error (results MS-179 in the IAG test plan).

Still open: the derotator drive has untuned velocity-loop defaults (0.097 / 8 ms); P-0-0165 cannot be compared from the CSV export (read it in the Drive Manager).
