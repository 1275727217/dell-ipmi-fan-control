# AGENTS.md

## Project

Single Bash script (`fan-control.sh`) that reads a Dell PowerEdge temperature
sensor via `ipmitool` and drives the fans on a tiered curve. When a sensor
name matches multiple readings (e.g. dual-CPU "Temp"), the highest value is
used. Runs as a systemd service or standalone daemon. No build step, no tests,
no linter.

## Key constraints

- **Bash 4+**, depends only on `ipmitool` and standard coreutils.
- Config file (`fan-control.conf`) is **sourced as Bash** — it sets shell
  variables and the `CURVE` array. Quote values; keep `CURVE` format
  `("temp:percent" ...)` ascending.
- `.gitignore` excludes `*.conf`; only `fan-control.conf.example` is tracked.
  Never commit real configs (they may contain iDRAC passwords).
- Compatibility: PowerEdge 11G–13G only. 14G+ blocks the raw IPMI fan commands.

## Safety invariants

- `set_fan_auto` (restore Dell automatic mode) MUST run on: `SIGTERM`,
  `SIGINT`, `SIGHUP`, `EXIT`, and any sensor-read failure (failsafe path).
- Do not remove or reorder the `trap cleanup` line.
- The PID-file guard prevents duplicate instances — keep it.

## Editing the script

- `usage()` extracts the header comment (lines 2–20) — keep that block in sync
  with actual subcommands.
- Subcommands are dispatched via `case "${1:-run}"` at the bottom.
- Main loop order: `get_temp` → `temp_to_percent` → `set_fan_speed` → `sleep`.
- Hysteresis logic in `temp_to_percent` prevents fan oscillation at tier
  boundaries — do not simplify without understanding the step-down guard.
