---
name: fmpy-simulate
description: Run an FMPy simulation for a given FMU file. Use when the user wants to simulate an FMU, run a Functional Mock-up Unit, or work with FMI-based models.
disable-model-invocation: false
user-invocable: true
argument-hint: <path-to-fmu> [start=0] [stop=10] [step=0.01] [out=/tmp/results.csv]
allowed-tools: Bash, Read, Glob
---

# FMPy FMU Simulation

> **What is an FMU?** A Functional Mock-up Unit (FMU) is a simulation model in a standardised format (FMI standard). It can represent control systems, physical models, or entire plants. FMPy is a Python tool for running these models.

## Step 1 — Parse arguments

Extract from `$ARGUMENTS`:
- `FMU_PATH` — path to the `.fmu` file (required, first positional arg)
- `start` — simulation start time in seconds (default: read from FMU, fallback `0.0`)
- `stop` — simulation stop time in seconds (default: read from FMU, fallback `10.0`)
- `step` — output interval / step size in seconds (default: read from FMU, fallback `0.01`)
- `out` — output CSV path (default: `/tmp/<fmu-name>_results.csv`)

If no FMU path is provided, ask the user for one. If the path doesn't end in `.fmu`, use the Glob tool to search nearby for `.fmu` files and suggest them.

## Step 2 — Run everything in one script

Use a single `uv run` invocation — this handles all dependencies automatically with no manual installation needed.

```bash
uv run --with fmpy --with matplotlib python3 - <<'PYEOF'
import sys, os
import numpy as np
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
from fmpy import read_model_description, dump, supported_platforms
from fmpy.simulation import simulate_fmu
import platform as _platform

FMU_PATH        = "$FMU_PATH"
start_arg       = "$start"   # empty string if not provided
stop_arg        = "$stop"
step_arg        = "$step"
OUTPUT_CSV      = "$out"

# ── 1. Check file exists ─────────────────────────────────────────────────────
if not os.path.isfile(FMU_PATH):
    print(f"ERROR: FMU file not found: {FMU_PATH}", file=sys.stderr)
    sys.exit(1)

# ── 2. Read model description ─────────────────────────────────────────────────
print("=" * 60)
print("FMU MODEL INFO")
print("=" * 60)
dump(FMU_PATH)

md = read_model_description(FMU_PATH)

# ── 2b. List tunable parameters ───────────────────────────────────────────────
params = [v for v in md.modelVariables if v.causality == 'parameter']
if params:
    print("\nTunable Parameters")
    print(f"  {'Name':<25} {'Start Value':>15}  {'Unit':<10}  Description")
    print("  " + "-" * 70)
    for v in params:
        val  = v.start if v.start is not None else "—"
        unit = v.unit  if v.unit  is not None else ""
        desc = v.description if v.description else ""
        print(f"  {v.name:<25} {str(val):>15}  {unit:<10}  {desc}")
else:
    print("\nNo tunable parameters found.")

# ── 3. Platform check ─────────────────────────────────────────────────────────
system = _platform.system().lower()
os_tag = "darwin64" if system == "darwin" else "linux64" if system == "linux" else "win64"

supported = supported_platforms(FMU_PATH)
if supported and os_tag not in supported and "c-code" not in supported:
    print(f"\nERROR: This FMU was built for {supported} and cannot run on {os_tag}.")
    print("You need a version of this FMU compiled for your operating system.")
    print("Ask the model author to provide a binary for your platform.")
    sys.exit(1)

# ── 4. Resolve simulation parameters from FMU defaults ───────────────────────
de = md.defaultExperiment
start_time      = float(start_arg) if start_arg else (float(de.startTime)  if de and de.startTime  is not None else 0.0)
stop_time       = float(stop_arg)  if stop_arg  else (float(de.stopTime)   if de and de.stopTime   is not None else 10.0)
output_interval = float(step_arg)  if step_arg  else (float(de.stepSize)   if de and de.stepSize   is not None else 0.01)

# ── 5. Choose simulation type ─────────────────────────────────────────────────
if md.coSimulation:
    sim_type = "CoSimulation"
elif md.modelExchange:
    sim_type = "ModelExchange"
else:
    print("ERROR: FMU supports neither CoSimulation nor ModelExchange.", file=sys.stderr)
    sys.exit(1)

# ── 6. Run simulation ─────────────────────────────────────────────────────────
print("\n" + "=" * 60)
print("SIMULATION")
print("=" * 60)
print(f"  Type     : {sim_type}")
print(f"  Start    : {start_time} s")
print(f"  Stop     : {stop_time} s")
print(f"  Step     : {output_interval} s")
print(f"  Steps    : {int((stop_time - start_time) / output_interval)}")
print(f"  Output   : {OUTPUT_CSV}")
print()

try:
    result = simulate_fmu(
        FMU_PATH,
        start_time=start_time,
        stop_time=stop_time,
        output_interval=output_interval,
        fmi_type=sim_type,
    )
except Exception as e:
    print(f"ERROR during simulation: {e}", file=sys.stderr)
    print("Suggestions:", file=sys.stderr)
    print("  - Try a larger step size (e.g. step=0.1)", file=sys.stderr)
    print("  - Check that the FMU binary matches your OS", file=sys.stderr)
    sys.exit(1)

# ── 7. Save CSV ───────────────────────────────────────────────────────────────
os.makedirs(os.path.dirname(OUTPUT_CSV) or ".", exist_ok=True)
np.savetxt(OUTPUT_CSV, result, delimiter=",",
           header=",".join(result.dtype.names), comments="")
print(f"Saved {len(result)} rows to: {OUTPUT_CSV}")

# ── 8. Summary statistics ─────────────────────────────────────────────────────
print("\n" + "=" * 60)
print("RESULTS SUMMARY")
print("=" * 60)
time_col = result.dtype.names[0]
outputs  = [n for n in result.dtype.names if n != time_col]

print(f"  {'Variable':<22} {'Min':>10} {'Max':>10} {'Final':>10}  Unit")
print("  " + "-" * 58)
unit_map = {v.name: (v.unit or "") for v in md.modelVariables}
for col in outputs:
    vals = result[col]
    unit = unit_map.get(col, "")
    print(f"  {col:<22} {vals.min():>10.4f} {vals.max():>10.4f} {vals[-1]:>10.4f}  {unit}")

# ── 9. Plot ───────────────────────────────────────────────────────────────────
n = len(outputs)
fig, axes = plt.subplots(n, 1, figsize=(11, 3.2 * n), sharex=True)
if n == 1:
    axes = [axes]

colors = plt.rcParams['axes.prop_cycle'].by_key()['color']
for i, (ax, col) in enumerate(zip(axes, outputs)):
    unit = unit_map.get(col, "")
    label = f"{col} ({unit})" if unit else col
    ax.plot(result[time_col], result[col], color=colors[i % len(colors)], linewidth=1.5)
    ax.set_ylabel(label)
    ax.grid(True, alpha=0.4)
    ax.set_title(col, fontsize=10, loc='right', color='gray')

axes[-1].set_xlabel("Time (s)")
model_name = md.modelName or os.path.basename(FMU_PATH).replace(".fmu", "")
fig.suptitle(f"{model_name}  —  FMI {md.fmiVersion}  {sim_type}", fontsize=13, y=1.01)
plt.tight_layout()

plot_path = OUTPUT_CSV.replace(".csv", ".png")
plt.savefig(plot_path, dpi=150, bbox_inches="tight")
print(f"\nPlot saved to: {plot_path}")
PYEOF
```

## Step 3 — Display the plot

After the script completes, use the Read tool to display the PNG image to the user:

```
Read: <plot_path>
```

Then give a brief plain-language interpretation of the results — what the outputs represent, whether the simulation looks healthy, and any notable behaviour (e.g. oscillations, settling, saturation).

## Step 4 — Handle errors

| Error | Friendly explanation to give the user |
|-------|----------------------------------------|
| FMU file not found | "The file wasn't found at that path. Let me search for .fmu files nearby…" then use Glob |
| Wrong platform | "This FMU was compiled for Windows/Linux and can't run on your Mac. Ask the model author for a macOS (darwin64) build." |
| Simulation crash | "The simulation failed — this can happen with a step size that's too large. Try adding `step=0.001` to use a finer time step." |
| `uv` not found | "uv isn't installed. Install it with: `curl -Lsf https://astral.sh/uv/install.sh | sh`" |
| FMI version error | "This FMU uses FMI version X, which may require a newer version of fmpy. Run: `uv run --with fmpy python3 -c \"import fmpy; print(fmpy.__version__)\"` to check." |

## Usage examples

```
/fmpy-simulate ./models/BouncingBall.fmu
/fmpy-simulate ./models/BouncingBall.fmu stop=5 step=0.001
/fmpy-simulate ./models/BouncingBall.fmu start=1 stop=20 out=./results/bb.csv
/fmpy-simulate /path/to/model.fmu stop=100 step=0.05
```
