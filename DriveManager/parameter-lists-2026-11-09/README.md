# Drive parameter lists, MONET/S, 2026-11-09

Complete parameter lists (Drive Manager export, UTF-16 CSV: IDN; name; actual; set; min; default; max) of the elevation and the
azimuth drive, exported by Tim at 17:16, after the MONETS update of the day (develop d6985f8, HalfBROT 0.14.0, MONETcommon 0.18.5)
and before the next power-on test. They are the reference for the motor tuning of that day.

State worth knowing:
- Elevation P-0-0165 (Wake&Shake) was found at factory defaults (50 % / 50 %, waiting time 150 ms, Kp 0.04) at 17:11 and set again
  before this export: command mode Wake and Shake, activation On enable request, first and second phase torque 60 %, first phase
  waiting time 500 ms, Kp 0.00 (see MotorTuning/WakeAndShake-findings.md). The values were not stored anywhere in the project.
- Velocity loop and position loop (elevation: S-0-0100 20000.000, S-0-0101 10.0 ms, S-0-0104 0.08; azimuth: 120000.000, 30.0 ms,
  0.08), current loop 20.2 V/A (el) and 40.5 V/A (az), velocity filter 0.150 ms: these differ from the repo's `Elevation Startup List.csv`
  / `Azimuth Startup List.csv`; the SoE init commands in MONETS.tsproj agree with the drives for current loop gain and velocity filter,
  but not for the velocity integral time (tsproj 8 ms).
- The derotator drive was not exported.
