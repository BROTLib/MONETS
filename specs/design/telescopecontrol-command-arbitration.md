# `FB_MonetTelescopeControl` command arbitration and `bInterrupted`

Source: `../../../MONETcommon` (sibling repo, checked out at `../MONETcommon` relative to this repo) — `FB_MonetTelescopeControl.TcPOU`, `EXTENDS FB_AltAzTelescopeControl`.

## Priority order

Commands are arbitrated by priority, highest first: `bPark` > `bGoHome` > `bGoto` > `bSlew` > `bTrack`. The arbitration `IF/ELSIF` chain (~line 200-300) checks the highest-priority active flag first; if a lower-priority command is currently in progress when a higher-priority one is requested, the lower one is cancelled and flagged as interrupted:

```
ELSIF bPark THEN
    bTrack := FALSE;
    IF bGoto OR bSlew OR bGoHome THEN
        bGoHome := FALSE;
        bGoto := FALSE;
        bSlew := FALSE;
        bInterrupted := TRUE;
        _HomeTelescope();
        _GotoTelescope();
        _SlewTelescope();
    ELSE
        IF bReady AND NOT bBusy THEN
            bInterrupted := FALSE;
            TCS_command := E_TCSCommand.park;
        END_IF
    END_IF
```

The same pattern repeats for `bGoHome` (interrupts `bGoto`/`bSlew`, additionally requires `bStopped` before proceeding) and `bGoto` (interrupts `bSlew`).

`bInterrupted := TRUE` is a one-shot signal: it's set, the lower-priority FSMs are each called once (so they see `bInterrupted` and reset their internal `nStage` to 0 — see below), then on the next scan the interrupting command's own flag (`bGoto`/`bSlew`/`bGoHome`) is already `FALSE`, so the arbitration falls into the `ELSE` branch, which is the **only place `bInterrupted` gets cleared** — gated on `bReady` (and `bStopped` for GoHome).

## How the individual FSMs react to `bInterrupted`

Every per-command state machine (`_GotoTelescope`, `_SlewTelescope`, `_HomeTelescope`, `_ParkTelescope`, `_PowerOn`) starts with:

```
IF bReset OR bError OR bInterrupted THEN
    nStage := 0;
    RETURN;
END_IF
```

So while `bInterrupted` is `TRUE`, every one of these FSMs is held at stage 0 and does nothing else that scan.

## The deadlock (see [ADR 0001](../adrs/0001-telescopecontrol-reset-must-be-driven-by-main.md))

`_PowerOn()` stage 10 is the only place that sets `fbElevation.Enable`/`fbAzimuth.Enable`/`fbDerotator.Enable := TRUE`. Because `_PowerOn()` is itself held at stage 0 whenever `bInterrupted` is true, and `bReady` (needed to clear `bInterrupted`) requires those axes to already be enabled, a `bInterrupted` left `TRUE` with no active command in flight cannot self-clear. Confirmed live on the running controller on 2026-09-09 — see `../plans/2026-09-09-static-review-and-live-ads-findings.md`.

## Open question

Should the arbitration logic self-heal (e.g. clear `bInterrupted` after N scans with `bError = FALSE` and no command flag active for that whole window), or should recovery always require an explicit `TelescopeControl.bReset`? Decided in ADR 0001 to require an explicit reset for now — revisit if this recurs often enough to be operationally annoying.
