# soak_pwm_equilibrium

A [Klipper](https://www.klipper3d.org/) macro that detects when a heated bed has
reached **thermal equilibrium** (a fully saturated "heat soak") by watching the
heater's PWM duty cycle instead of guessing with a fixed timer.

Once a bed holds its target temperature, the heater PWM settles to a steady,
low value as it only replaces heat lost to the environment. `soak_pwm_equilibrium`
samples that PWM into a rolling 5-minute window and reports a `SOAKED=True/False`
verdict when the duty cycle has stopped falling and the plate is at temperature.

## Why PWM instead of a timer

A fixed "soak for N minutes" delay is either too short (plate not saturated) or
wastefully long. The heater PWM is a direct proxy for how much heat the plate is
still absorbing: while it keeps dropping, the bed is still soaking; when it flattens
out at target, the plate is thermally saturated. This macro watches that curve and
tells you when equilibrium is actually reached.

## Contents

| File | Purpose |
|------|---------|
| `heatsoak.cfg` | The heat-soak monitor: rolling temperature/PWM sampler, soak verdict, and one-shot query macros. Persists its buffer across short reboots. |

## Requirements

- **Klipper** (recent release — uses Jinja2 templating and the `printer` object
  fields `heater_bed.power`, `webhooks.state`, and `save_variables.variables`).
- A web front end (**Mainsail** or **Fluidd**) is recommended for running the
  macros and reading `action_respond_info` output in the console.
- The `[save_variables]` feature must be enabled so the rolling buffer can
  survive a brief restart:

  ```ini
  [save_variables]
  filename: ~/printer_data/config/variables.cfg
  ```

## Installation

1. Place `heatsoak.cfg` where Klipper can read it, typically under
   `~/printer_data/config/`. For example:

   ```
   ~/printer_data/config/soak_pwm_equilibrium/heatsoak.cfg
   ```

2. Include it from your main `printer.cfg` (path is relative to the directory
   containing `printer.cfg`):

   ```ini
   [include soak_pwm_equilibrium/heatsoak.cfg]
   ```

3. Add the `[save_variables]` prerequisite above if it is not already present.

4. Restart Klipper (`RESTART`, then `FIRMWARE_RESTART` if the config change
   requires it) and check the console for errors.

## Usage

The sampler auto-starts on boot via a `delayed_gcode`. Available commands:

| Command | Description |
|---------|-------------|
| `SOAK_MONITOR_START` | Begin sampling (runs automatically on boot). |
| `SOAK_MONITOR_STOP` | Stop sampling — call from your cleanup / `CANCEL_PRINT`. |
| `SOAK_STATUS` | Print the 5-minute average temperature, average/early/late PWM, and a `SOAKED=True/False` verdict. |
| `QUERY_BED_PWM` | One-shot read of current bed PWM duty cycle and temperature. |

Typical flow: heat the bed to target, then poll `SOAK_STATUS` (or watch it in a
macro loop) until it reports `SOAKED=True` before starting your print.

## How it works

- **Sampling.** Every `sample_period` seconds a `delayed_gcode` loop records the
  bed temperature and `heater_bed.power` (PWM, 0.0–1.0) into rolling buffers
  trimmed to the `window_seconds` window (default 5 minutes → 60 samples at 5s).
- **Verdict.** `SOAK_STATUS` splits the window into an early and a late half. If
  the PWM drift between halves is under `pwm_tolerance` (stable) and the minimum
  temperature is within `temp_band` of target (at temp), it reports `SOAKED=True`.
- **Reboot survival.** The buffer and a timestamp are persisted via
  `[save_variables]` on every sample. On boot, history is restored only if the
  restart gap looks short (`stale_after_seconds`); otherwise it starts clean so a
  cooled-down bed is never reported as soaked.

## Tunables

Set at the top of `heatsoak.cfg`:

| Variable | Default | Meaning |
|----------|---------|---------|
| `sample_period` | `5` | Seconds between samples. |
| `window_seconds` | `300` | Rolling window length (5 minutes). |
| `stale_after_seconds` | `120` | Discard restored history if the boot gap exceeds this. |
| `pwm_tolerance` | `0.02` | Max PWM drift between window halves to count as stable (<2%). |
| `temp_band` | `1.0` | Degrees C below target still counted as "at temp". |

## Notes & caveats

- `estimated_print_time` is a monotonic clock that resets on each Klipper
  restart; it cannot by itself measure wall-clock downtime. The reboot-survival
  heuristic is deliberately conservative. If you have an RTC or a host script
  recording real time, prefer that for the staleness check.
- Runtime state stored in `gcode_macro` variables is marked "do not edit" in the
  config — only change the tunables.

## License

See [LICENSE](LICENSE).
