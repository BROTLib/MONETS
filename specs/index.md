# MONETS specs

Living documentation for the MONETS TwinCAT firmware, split by purpose:

- **[adrs/](adrs/index.md)** — Architecture Decision Records. Short, numbered docs capturing a single decision and why it was made, so the reasoning survives even after the code around it changes.
- **[design/](design/index.md)** — Living design notes per subsystem or interface. Updated in place as understanding evolves; not tied to a single point in time.
- **[plans/](plans/index.md)** — Dated investigation/work logs, one per bug fix, feature, or review. The day-to-day record of what was found and why a change was made.

Loosely modeled on [pyobs-core's `specs/`](https://github.com/pyobs/pyobs-core/tree/main/specs) convention, scaled down for a single-controller PLC firmware repo (no `steering/` — MONETS is one codebase, not a multi-project fleet).
