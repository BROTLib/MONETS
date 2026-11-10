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
