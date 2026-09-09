# 0001. `TelescopeControl.bReset` must be driven from `MAIN`, not left to sub-FBs

## Status

Proposed (2026-09-09) — not yet implemented, see [BROTLib/MONETS#3](https://github.com/BROTLib/MONETS/issues/3).

## Context

`FB_MonetTelescopeControl.bReset` is the only code path that unconditionally clears a latched `bInterrupted` state (it forces `bInterrupted := FALSE` regardless of `bReady`, see `specs/design/telescopecontrol-command-arbitration.md`). Everything else that could clear `bInterrupted` is gated on `bReady`, which itself depends on the axes being enabled — and axis re-enable depends on `bInterrupted` already being clear. This is a real deadlock: once `bInterrupted` latches with no command in flight, nothing inside `FB_MonetTelescopeControl` can recover on its own.

`MAIN.TcPOU`'s cabinet/pendant reset handler was found to be dead code (`IF CabinetControl.IsResetPushed() OR PendantControl.IbResetButton THEN ; END_IF`) — confirmed live on the running controller on 2026-09-09 that `bInterrupted` was stuck `TRUE` with no way to clear it.

## Decision

`MAIN.TcPOU` is the only place that should ever set `TelescopeControl.bReset`. It must translate the physical/software reset request (cabinet button, pendant button, or a future MQTT `reset` command) directly into `TelescopeControl.bReset := TRUE`, plus explicit `.Reset()` calls on `CoverControl` and `HydraulicsControl` (which are separate FBs, not reachable through `TelescopeControl`'s internal cascade).

Reasoning for putting this in `MAIN` rather than inside `FB_MonetTelescopeControl` itself: the reset is a physical operator action tied to cabinet/pendant hardware, which only `MAIN` has visibility into. Making `FB_MonetTelescopeControl` self-heal `bInterrupted` (e.g. auto-clear it after N scans with no active command) was considered but rejected for v1 — a silent auto-recovery could mask the underlying cause of why a command was interrupted and never resumed, whereas a manual reset gives an operator a deliberate, visible recovery action.

## Consequences

- The physical reset button must actually do something once this is implemented — verify on hardware, not just via ADS, since a mis-wired `IsResetPushed()`/`IbResetButton` read was part of why this went unnoticed.
- `FB_MonetCoverControl.Reset()` has an independent bug (sets `Reset := TRUE`, its own return value, instead of `bReset := TRUE`) that must be fixed in the same change or the cascade above silently does nothing for cover errors.
