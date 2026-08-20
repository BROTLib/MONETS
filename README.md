# MONETS

TwinCAT 3 control application for **MONET/S**, the 1.2 m Alt-Az robotic
telescope of the MONET network (Georg-August-Universität Göttingen), located at
the **South African Astronomical Observatory (SAAO), Sutherland, South Africa**
(lon 20.810808°, lat −32.375823°, alt 1798 m).

The application is nearly identical to **MONETN** (same mount, drives and
safety hardware) but configured for the Southern Hemisphere site. It controls
the telescope mount (Azimuth / Elevation / Derotator axes), focus, the three
mirror covers, the hydraulic pump/brake system, the observatory roof, power
monitoring, cabinet I/O and the TwinSAFE safety chain, and publishes
MQTT/InfluxDB telemetry. It is built on the **BROTLib** core library, the
**HalfBROT** hardware layer and the **MONET Roof** library (`FB_RoofControl`),
and — unlike MONETN — consumes the **MONETcommon** library **directly**: the
MONETS project contains almost no local function blocks beyond `MAIN` and two
site-specific POUs.

### Name and heritage ("STELLA1")

Like MONETN, the project descends from the **STELLA1** telescope control
application (STELLA is the sister AIP/Göttingen robotic telescope); the stale
`Mappings.xml` still uses the old PLC instance name
`TIPC^STELLA1Runtime^STELLA1Runtime Instance`. The legacy README stub
(`# STELLA1 / Configuration`: PLC `CX-7A03E9`, AMS Net-ID `5.122.3.233.1.1`,
IP `161.72.132.76`) is **out of date / only partially verifiable** — the PLC
device name and IP do not appear in this repository; the checked-in project
uses the target Net-ID `10.1.180.77.1.1` (development machine).

---

## Repository layout

```
MONETS/
├── MONETS.sln                  # TwinCAT solution (DriveManager, MotorTuning, Scope, MONETS)
├── Azimuth Startup List.csv    # SoE parameter startup list – azimuth AX5125
├── Elevation Startup List.csv  # SoE parameter startup list – elevation AX5125
├── DriveManager/               # TwinCAT Drive Manager 2 project + .tcdmdrv exports
│   ├── DriveManager.tcdmproj
│   ├── Azimuth (AX5125-0000-0214).tcdmdrv
│   ├── Elevation (AX5125-0000-0214).tcdmdrv
│   └── Derotator (AX5206-0000-0215).tcdmdrv
├── MotorTuning/                # TwinCAT ScopeView motor-tuning projects
├── Scope/                      # TwinCAT ScopeView projects
├── MONETS/
│   ├── MONETS.tsproj           # TwinCAT system project (TwinCAT 3.1.4024.66)
│   ├── Mappings.xml            # Stale mapping file (STELLA1Runtime; old names)
│   ├── MONETSRuntime/          # PLC project (current)
│   │   ├── MONETSRuntime.plcproj  # references MONETcommon directly
│   │   ├── PlcTask.TcTTO       # PLC task (10 ms, priority 20, calls MAIN)
│   │   ├── POUs/               # MAIN, FB_TelescopeAuxiliary, FB_WeatherCheck
│   │   └── VISUs/              # TwinCAT visualizations (11 screens)
│   └── MONETSTwinSAFE/         # Safety project (TwinSAFE group on EL6910)
└── README.md
```
(No `_Boot/` or `_Libraries/` folders are checked in; the `.tmc` files are
git-ignored.)

---

## Hardware / EtherCAT topology

Same hardware family as MONETN — two EtherCAT buses:

**Device 4** (mount drives + roof bus): **Azimuth** and **Elevation** as
`AX5125-0000-0214` (SoE; **ETEL TMB 0530-070-3VDN** resp. **ETEL
TMB 0360-070-3XBS** torque motors), **Derotator** as `AX5206-0000-0215`
(Beckhoff **AM8532** motor, **SICK EKM36** absolute encoder). Each drive
carries an **AX5805** TwinSAFE safety card (STO). Term 11 (EK1100) starts the
roof bus with the same topology as the MONETRoof application (EL2008
direction/reset outputs, EL1008 counter/limit/fault inputs, EL4004 speed
setpoints, EL1904/EL2904 TwinSAFE I/O).

**Device 1** (telescope I/O bus, coupler Term 8 EK1200): EL6910 TwinSAFE PLC
(FSoE master), EL1904/EL2904 safety I/O, EL3483 3-phase power monitoring,
EL5101/EL7342 focus (Faulhaber motor), EL3202/EL3164 temperature inputs,
EL2008/EL1008 hydraulics, mirror-cover I/O (EL1008/EL2004), front panel
(EL1008/EL2004), pendant I/O and EL9410 power supply.

**NC axes**: `Fokus`, `Azimuth`, `Elevation`, `Derotator`.

---

## PLC application architecture

The PLC task (`PlcTask`, 10 ms, priority 20) calls `MAIN`, which instantiates
the MQTT communication function block and the subsystem controllers (from the
MONETcommon library and HalfBROT):

```
MAIN
├── fbComm             : FB_Comm_MQTT_Influx          (MQTT + Influx telemetry, BROTLib)
├── SafetyHandling     : FB_MonetSafetyHandling       (MONETcommon)
├── CabinetControl     : FB_MonetCabinetControl       (MONETcommon)
├── PowerMonitoring    : FB_MonetPowerMonitoring      (MONETcommon)
├── RoofControl        : FB_RoofControl               (MONET Roof library)
├── CoverControl       : FB_MonetCoverControl         (MONETcommon)
├── HydraulicsControl  : FB_MonetHydraulicsControl    (MONETcommon)
├── FocusControl       : FB_MonetFocusControl         (MONETcommon)
├── DerotatorControl   : FB_DerotatorControl          (HalfBROT)
├── ElevationControl   : FB_ElevationControl          (HalfBROT)
├── AzimuthControl     : FB_AzimuthControl            (HalfBROT)
├── TelescopeControl   : FB_MonetTelescopeControl     (MONETcommon)
├── PendantControl     : FB_MonetPendantControl       (MONETcommon)
├── TelescopeAuxiliary : FB_TelescopeAuxiliary        (site-specific)
└── WeatherCheck       : FB_WeatherCheck              (site-specific)
```

`MAIN` wires the site configuration (`ST_TelescopeConfig`:
`name := 'MONET/S'`, `diameter := 1.2`, `mount := 'ALT_AZ'`, park positions
`azimuthPark := 175`, `elevationPark := 60`), derives the telescope mode from
the cabinet key switch (off / manual / automatic) and performs MQTT host
discovery via `FB_GetHostName`.

### Site-specific POUs

- `FB_WeatherCheck` — polls the site weather station over HTTP and feeds the
  weather data into the control logic (currently disabled in the shipped
  configuration).
- `FB_TelescopeAuxiliary` — mirror/cell/flange temperature sensors and other
  auxiliary telescope functions.

---

## States and commands

Identical to MONETN: telescope modes `E_TelescopeMode` (off / manual /
automatic); TCS commands `E_TCSCommand` (`poweron > stop > park > gohome >
goto > slew > track > no_command`) as stage-based state machines with
per-command timeouts; derived states `bHomed`, `bReady`, `bTracking`,
`bIsParked`, `fReadyState`, `nMotionState`; auto-park after 12 h; roof with
automatic/manual/slow modes and per-roof commands.

## Communication and telemetry

`FB_Comm_MQTT_Influx` (BROTLib) connects to the MQTT broker and publishes
telemetry in Influx line protocol:

| Parameter | Value |
|---|---|
| Broker host | `192.168.127.10` |
| Port | `1883`, keep-alive 60 s |
| Subscribe topic (commands) | `MONETS/Telescope/SET` |
| Publish topic (telemetry) | `MONETS/Telemetry` |
| Log topic | `MONETS/Log` |

In addition to the 12-hour auto-park timeout, MONETS runs an **MQTT watchdog**
(`mqttWatchdog`, 30 s grace period): if the broker connection is lost for more
than the timeout, the telescope is parked and the roof is closed automatically.

## Safety (TwinSAFE)

A TwinSAFE application (`MONETSTwinSAFE`, group `TwinSafeGroup1`) runs on the
**EL6910** safety PLC (FSoE master): E-stop (`FBEstop1`, safeEstop) → **STO on
all three drive axes** via the AX5805 cards; external device monitoring
(`FBEdm*`, safeEdm) on the mirror-cover, hydraulics and roof contactors;
per-axis STO state/reset alias devices
(`Azimuth/Elevation/Derotator_STOState`, `*_STOReset`); group ports `Run`,
`Restart`, `ErrorAcknowledgement` and status ports. The PLC side
(`FB_MonetSafetyHandling`) performs the startup sequence (Run/ErrAck/Restart/
STO resets).

## Visualization

Eleven TwinCAT visualization screens provide the operator HMI: the main
`Visualization` overview, `Telescope`, `Azimuth`, `Elevation`, `Derotator`,
`Focus`, `Cover`, `Hydraulics`, `Pendant`, `Roof` and `Safety`.

## Configuration

- **Drive startup lists** (`Azimuth/Elevation Startup List.csv`): SoE parameter
  startup lists for the AX5125 drives (identical to MONETN — the two telescopes
  share the same Halfmann mount hardware); derotator configured via DriveManager
  only (`AX5206-0000-0215`).
- **DriveManager** (`DriveManager.tcdmproj`): the three `.tcdmdrv` exports.
- **MotorTuning/Scope**: TwinCAT ScopeView projects for motor tuning and
  commissioning.
- **MAIN constants**: `telescopeConfig` (see above), roof parameters, axis
  ranges and pointing-model coefficients (MONETcommon).

## Dependencies

- **BROTLib**, **AstroBROT** (BROT) — core library, communication, telescope
  control base classes, tracking/pointing functions, shared DUTs.
- **HalfBROT** (BROT) — axis and hardware function blocks.
- **MONETcommon** (IAG) — the MONET control logic, referenced directly
  (`PlaceholderReference Include="MONETcommon"`).
- **MONET Roof** (`MONET_Roof`) — `FB_RoofControl`/`I_Roof`.
- Beckhoff system libraries: `Tc2_MC2`, `Tc2_MC2_Drive`, `Tc2_NC`,
  `Tc3_IotBase`/`Tc3_IotCommunicator` (MQTT), `Tc2_Standard`, `Tc2_System`,
  `Tc2_Utilities`, `Tc3_Module`, plus the TwinCAT visualization libraries.

## Unification with MONETcommon

MONETS is the reference consumer of the MONETcommon library and represents the
target state of the code: site-specific logic lives in `MAIN` plus the two
site-specific POUs. The unification plan
([BROTLib/MONET_Unification.md](../BROTLib/MONET_Unification.md)) uses MONETS
as the reference for migrating MONETN away from its vendored copies.

## Building and deployment

The solution is built with TwinCAT 3.1 in TwinCAT XAE (system project
3.1.4024.66, POU files 3.1.4024.15, PLC project authored with 3.1.4026.13;
configurations for TwinCAT RT (x64/x86), CE7 (ARMV7) and TwinCAT OS). The PLC
project is `MONETSRuntime` (ADS port 851, symbolic mapping), task `PlcTask`
10 ms/priority 20; NC-Task 1 SAF 2 ms / SVB 10 ms. MQTT requires the Tc3 IoT
license. No boot project is checked in; the controller boots from TwinCAT's
own boot project on the CX.
