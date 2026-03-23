---
name: plecs-sim-autotune
description: Use when the user wants to automate PLECS circuit simulation, auto-tune PID/compensator parameters for power converters, modify .plecs model parameters programmatically, run closed-loop simulations via XML-RPC, analyze transient response (overshoot/undershoot/oscillations/settling time), or design Type 3 compensators. Trigger phrases include "tune PID", "auto-tune", "run PLECS simulation", "compensator design", "adjust Kp Ki Kd Kf", "overshoot undershoot", "buck converter tuning", "sweep parameters", "transient analysis".
version: 1.0.0
---

# PLECS Simulation Automation & PID Auto-Tuning

This skill automates PLECS power-converter simulation and closed-loop PID tuning via XML-RPC, without any manual GUI interaction.

## When This Skill Applies

- User wants to run PLECS simulations programmatically
- User wants to tune PID/compensator gains for a power converter
- User wants to modify .plecs model parameters and re-simulate in a loop
- User wants to analyze transient response metrics (OS, US, oscillations, settling time)
- User wants to design a Type 3 compensator from crossover frequency and phase margin

## Architecture Overview

```
User request
    |
    v
[CompensatorDesign]  <-- analytical: (wc, phi_m) -> (Kp, Ki, Kd, Kf)
    |
    v
[PlecsModelEditor]   <-- writes Kp/Ki/Kd/Kf into .plecs XML file
    |
    v
[PlecsRpc]           <-- XML-RPC: load model, run sim, export scope CSV
    |
    v
[ResponseAnalyzer]   <-- parse CSV, measure OS/US/oscillations/settling
    |
    v
[PidTuner]           <-- rule-based 2D search: adjust wc, phi_m
    |
    v
(loop until PASS or max iterations)
```

## PLECS XML-RPC Connection

### Endpoint
```
http://127.0.0.1:1080/RPC2
```

### Prerequisites
- PLECS Standalone must have XML-RPC enabled
  - Windows registry: `HKCU\Software\Plexim\PLECS4.9\XmlRpcEnabled` = `true`
  - Or for PLECS 5.0: check equivalent registry key
- Python `xmlrpc.client` (standard library, no install needed)

### Connection Pattern
```python
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://127.0.0.1:1080/RPC2', allow_none=True)
# Test connection:
server.plecs.statistics()
```

### Essential RPC Calls

| Call | Purpose |
|---|---|
| `plecs.statistics()` | Check connection, list loaded models |
| `plecs.load(path)` | Load a .plecs model file (absolute path) |
| `plecs.close(model_id)` | Close a loaded model |
| `plecs.webserver('startSimulation', model_id, {})` | Start simulation |
| `plecs.webserver('getSimulationState', model_id, {})` | Poll until not "running" |
| `plecs.webserver('getScopeCsv', 'model_id/Scope', {})` | Export scope data as CSV bytes |
| `plecs.webserver('getScopeBitmap', 'model_id/Scope', {})` | Export scope as PNG bitmap |
| `plecs.webserver('getScopeInfo', 'model_id/Scope', {})` | Get scope channel info |

### Model ID
The `model_id` is the filename stem without `.plecs` extension. For `synchronous buck.plecs`, the model_id is `"synchronous buck"`.

### Decoding RPC Binary Data
`getScopeCsv` returns an XML-RPC Binary object. Extract the bytes:
```python
res = server.plecs.webserver('getScopeCsv', f'{model_id}/Scope', {})
blob = res['csv'] if isinstance(res, dict) else res
raw_bytes = blob.data if hasattr(blob, 'data') else blob
```

## Modifying .plecs Model Parameters

PLECS model files are XML-like text. Parameters can be modified with regex substitution:

```python
import re

with open(plecs_path, 'r', encoding='utf-8') as f:
    content = f.read()

# Replace parameter values
content = re.sub(r'Ki\s*=\s*[\d.e+-]+;', f'Ki = {Ki};', content)
content = re.sub(r'Kf\s*=\s*[\d.e+-]+;', f'Kf = {Kf};', content)
content = re.sub(r'Kd\s*=\s*[\d.e+-]+;', f'Kd = {Kd};', content)

# Kp may be split across a quoted string boundary in some .plecs files:
#   Kp"\n" = value;
# Try normal pattern first, then fallback:
if re.search(r'Kp\s*=\s*[\d.e+-]+;', content):
    content = re.sub(r'Kp\s*=\s*[\d.e+-]+;', f'Kp = {Kp};', content)
else:
    content = re.sub(r'Kp"\n"\s*=\s*[\d.e+-]+;', f'Kp = {Kp};"\n"', content)

# Write to a TEMP copy — never overwrite the original
with open(work_dir / "temp_model.plecs", 'w', encoding='utf-8') as f:
    f.write(content)
```

**Important:** Always write to a temp copy in a work directory, then load that copy via RPC. Never overwrite the user's original `.plecs` file.

## Simulation Loop Pattern

```python
# 1. Modify parameters in .plecs file
modified_path = editor.modify_params(Kp, Ki, Kd, Kf)

# 2. Close any existing model, load the modified one
rpc.load_model(modified_path)

# 3. Run simulation
rpc.start_simulation()
rpc.wait_simulation_done(poll_interval=0.2, timeout=60.0)

# 4. Export scope data
csv_bytes = rpc.get_scope_csv(f'{model_id}/Scope')

# 5. Parse and analyze
header, data = parse_csv(csv_bytes)
overshoot, undershoot, osc_count, settling_time = analyze(header, data)
```

## Type 3 Compensator Design

### Plant: Buck Converter Control-to-Output Transfer Function

```
Gvd(s) = Vdc * (1 + s*Rc*Cout) / (1 + s*(Rc+Rl)*Cout + s^2*L*Cout)
```

Key frequencies for a 12V-to-5V buck (L=30uH, C=15uF, Rc=7.5mOhm, Rl=50mOhm):
- LC resonance: w0 = 1/sqrt(LC) = 47,140 rad/s (7.5 kHz)
- ESR zero: wesr = 1/(Rc*Cout) = 8,888,889 rad/s (1.4 MHz)
- Quality factor: Q = sqrt(L/C) / (Rc+Rl) = 24.6

### Design Variables (2D search space)

| Variable | Symbol | Range | Unit |
|---|---|---|---|
| Crossover frequency | wc | 94,248 to 314,159 | rad/s (15-50 kHz) |
| Phase margin | phi_m | 0.5236 to 1.3963 | rad (30-80 deg) |

### Analytical Equations: (wc, phi_m) -> (Kp, Ki, Kd, Kf)

```python
wl = wc / 10.0                              # integrator pole, decade below crossover
Gvd_wc = Gvd(j * wc)                        # evaluate plant at crossover
gain_plant = abs(Gvd_wc)
phi_plant = phase(Gvd_wc)

phi_boost = (-pi + phi_m) - phi_plant        # phase compensator must add
phi_boost = clamp(phi_boost, -89.4 deg, +89.4 deg)

wz = wc * sqrt((1 - sin(phi_boost)) / (1 + sin(phi_boost)))   # compensator zero
wp = wc * sqrt((1 + sin(phi_boost)) / (1 - sin(phi_boost)))   # compensator pole

Gpid0 = (1/gain_plant) * sqrt((1 + (wc/wp)^2) / (1 + (wc/wz)^2))

Ki = Gpid0 * wl
Kf = wp
Kp = Gpid0 * (1 + wl/wz) - Ki/Kf
Kd = max(0, Gpid0/wz - Kp/Kf)              # clamp non-negative
```

**Reference design** (wc = 25 kHz, phi_m = 60 deg):
Kp = 0.31703, Ki = 3764.63, Kd = 4.785e-6, Kf = 551,786

### Why wc_min = 2 * w0

Near LC resonance (w0), the plant phase drops to -180 deg, causing phi_boost to approach +90 deg. At +90 deg, `sin(phi_boost) = 1`, making `wz -> 0` and `wp -> infinity` — a singularity. Setting `wc_min = 2*w0` keeps the equations numerically stable.

### Why Kd is Clamped to 0

When wc is low (near resonance), the equations can produce negative Kd. In PLECS, the PID block computes `kbc = sqrt(Kd/Ki)` internally — negative Kd makes this imaginary and crashes the simulation. Clamping `Kd = max(0, Kd)` produces a Type 2 compensator (PI + filter) instead, which is still valid.

## Transient Response Analysis

### Metrics Extracted from Raw Vout Waveform

```
v_peak   = max(Vout[step_idx : step_idx + 3ms])    # peak after load step
v_valley = min(Vout[step_idx : step_idx + 3ms])     # valley after load step

overshoot  = max(0, (v_peak   - Vref) / Vref * 100) %
undershoot = max(0, (Vref - v_valley) / Vref * 100)  %
```

Where `Vref` is the nominal output (e.g. 5.0V). Raw (unfiltered) data is used to capture true peak/valley including switching ripple.

### Oscillation Counting (Ripple-Immune)

Switching ripple at 250 kHz can produce false peaks. To avoid this:

1. Apply a moving-average filter: window = 4 switching cycles (16 us)
2. Find local maxima/minima with a wide neighborhood: 1/4 LC period (~33 us, ~580 samples)
3. Deduplicate: merge detections within 1/2 LC period, keep most extreme
4. Threshold: only count peaks/valleys deviating > 1% (50 mV) from Vref

### Load Step Detection

The load step time is detected from the inductor current (IL) waveform by finding a step edge (large dI/dt). The analysis window starts at the detected step and extends 3 ms forward.

## Tuning Strategy (Rule-Based 2D Search)

The tuner adjusts wc and phi_m based on failure mode:

| Failure Mode | wc Action | phi_m Action | Reasoning |
|---|---|---|---|
| Oscillations | decrease | increase | More phase margin damps resonance |
| High overshoot | decrease aggressively | — | Lower bandwidth, less overshoot |
| Moderate overshoot | decrease slightly | increase slightly | Phase-margin-limited |
| Undershoot only | increase | — | Higher bandwidth, faster recovery |
| Both OS and US | — | increase | Near-resonance, add damping |

### Anti-Stall Mechanisms
- **Decay factor:** step sizes scale as `max(0.2, 1 - 0.05*iter)`, reducing aggressiveness over iterations
- **Stall detection:** after 5 consecutive non-improvements, revert to best-known (wc, phi_m) + small perturbation
- **Resonance escape:** if stuck at wc_min with oscillations, jump to `2 * wc_min`

### Score Function
```
score = max(0, OS - target_OS) + max(0, US - target_US) + max(0, osc - max_osc) * 3
```
Oscillations weighted 3x heavier — they indicate fundamental stability issues.

### Pass Criteria
```
OS < target_OS AND US < target_US AND osc <= max_osc
```

## File Organization

```
project_root/
  auto_tune.py          # Main auto-tuner: all classes (PlecsRpc, CompensatorDesign, PidTuner, etc.)
  analyze.py            # Post-run analysis: plot_animation(), plot_metrics(), OS/US visualization
  gui.py                # PyQt5 GUI: live waveform, PID controls, pause/resume, GIF export
  synchronous buck.plecs # PLECS model file (original, never modified directly)
  plecs_tuning_work/    # Temp directory for modified .plecs copies during tuning
  results/              # Output: iter_000.csv ... iter_N.csv, tuning_log.csv, animation.gif
```

## Common Pitfalls

1. **Model ID with spaces**: `"synchronous buck"` not `"synchronous_buck"` — must match the .plecs filename stem exactly
2. **Close before reload**: Always `plecs.close(model_id)` before `plecs.load()` — otherwise PLECS may use the cached old model
3. **Regex for Kp**: The `Kp` parameter name sometimes spans a line break in .plecs XML (`Kp"\n"`). Handle both patterns.
4. **Simulation timeout**: Some parameter combinations cause very slow convergence. Set a timeout (60s default).
5. **wc near w0**: The compensator equations blow up near LC resonance. Enforce `wc >= 2 * w0`.
6. **Negative Kd**: Clamp to 0. Never pass negative Kd to PLECS.

## Quick Start: Run Auto-Tune

```bash
# Ensure PLECS is running with XmlRpc enabled, then:
python auto_tune.py

# Or launch the GUI:
python gui.py
```

## Quick Start: Single Simulation via RPC

```python
import xmlrpc.client

server = xmlrpc.client.ServerProxy('http://127.0.0.1:1080/RPC2', allow_none=True)
server.plecs.statistics()                                          # verify connection
server.plecs.load(r'C:\path\to\model.plecs')                      # load model
server.plecs.webserver('startSimulation', 'model_name', {})        # run
# poll until done:
import time
while 'running' in str(server.plecs.webserver('getSimulationState', 'model_name', {})).lower():
    time.sleep(0.2)
# export results:
res = server.plecs.webserver('getScopeCsv', 'model_name/Scope', {})
csv_bytes = res['csv'].data if hasattr(res['csv'], 'data') else res['csv']
with open('output.csv', 'wb') as f:
    f.write(csv_bytes)
```
