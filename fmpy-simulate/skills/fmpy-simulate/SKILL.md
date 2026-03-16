---
name: fmpy-simulate
description: Run an FMPy simulation for a given FMU file. Use when the user wants to simulate an FMU, run a Functional Mock-up Unit, or work with FMI-based models. Also handles follow-up messages from the simulation config widget (messages starting with "SIMULATE:").
disable-model-invocation: false
user-invocable: true
argument-hint: <path-to-fmu>
allowed-tools: Bash, Read, Glob
---

# FMPy FMU Simulation

> **What is an FMU?** A Functional Mock-up Unit (FMU) is a self-contained simulation model in the FMI standard format. It can represent physical systems, control loops, or entire plants. FMPy is a Python tool for running these models without installing any simulator software.

---

## Phase detection

Check `$ARGUMENTS`:

- If it starts with `SIMULATE:` → this is a **Phase 2** call from the widget button. Jump to **Phase 2**.
- Otherwise → this is a **Phase 1** call. Proceed below.

---

## Phase 1 — Inspect FMU and show configuration widget

### Step 1 — Resolve FMU path

Extract `FMU_PATH` as the first argument. If missing or not ending in `.fmu`, use the Glob tool to find `.fmu` files nearby and ask the user to pick one.

### Step 2 — Inspect the FMU

```bash
uv run --with fmpy python3 - <<'PYEOF'
import sys, os, json
from fmpy import read_model_description, dump, supported_platforms
import platform as _platform

FMU_PATH = "$FMU_PATH"

if not os.path.isfile(FMU_PATH):
    print(f"ERROR: File not found: {FMU_PATH}"); sys.exit(1)

dump(FMU_PATH)
md = read_model_description(FMU_PATH)

# Platform check
system = _platform.system().lower()
os_tag = "darwin64" if system == "darwin" else "linux64" if system == "linux" else "win64"
sp = supported_platforms(FMU_PATH)
if sp and os_tag not in sp and "c-code" not in sp:
    print(f"\nERROR: FMU supports {sp}, cannot run on {os_tag}."); sys.exit(1)

# Collect variables by role
de    = md.defaultExperiment
inputs  = [v for v in md.modelVariables if v.causality == 'input']
params  = [v for v in md.modelVariables if v.causality == 'parameter']
outputs = [v for v in md.modelVariables if v.causality == 'output']

sim_type   = "CoSimulation" if md.coSimulation else "ModelExchange"
start_time = float(de.startTime) if de and de.startTime is not None else 0.0
stop_time  = float(de.stopTime)  if de and de.stopTime  is not None else 10.0
step_size  = float(de.stepSize)  if de and de.stepSize  is not None else 0.01

info = {
    "fmu_path": FMU_PATH,
    "model_name": md.modelName,
    "fmi_version": md.fmiVersion,
    "sim_type": sim_type,
    "start": start_time,
    "stop": stop_time,
    "step": step_size,
    "inputs": [{"name": v.name, "start": v.start, "unit": v.unit or "", "desc": v.description or ""} for v in inputs],
    "params": [{"name": v.name, "start": v.start, "unit": v.unit or "", "desc": v.description or ""} for v in params],
    "outputs": [{"name": v.name, "unit": v.unit or "", "desc": v.description or ""} for v in outputs],
}
print("FMUINFO:" + json.dumps(info))
PYEOF
```

Capture the line starting with `FMUINFO:` and parse the JSON.

### Step 3 — Discuss the model

Before showing the widget, give the user a short plain-language introduction:
- What does this model simulate?
- What are the inputs and tunable parameters — what do they control physically?
- What will the outputs show?
- Any tips (e.g. "coefficient of restitution between 0 and 1 — higher means bouncier")

Keep it concise: 3–6 sentences.

### Step 4 — Generate and show the configuration widget

Generate the config widget HTML and call `show_widget` with it. If `show_widget` is not available, save to `/tmp/<model>_config.html` and open it with `open`.

The widget must:
- Show model name, FMI version, sim type
- Have number inputs for **start**, **stop**, **step** (pre-filled with FMU defaults)
- Have number inputs for every **input variable** and **tunable parameter** (pre-filled with their start values)
- Show unit and description next to each field
- Have a prominent **▶ Run Simulation** button
- On click, call `sendPrompt()` with a message encoding all values

```python
import json

# (use the parsed info dict from Step 2)
info = <parsed FMUINFO dict>

fmu_path   = info["fmu_path"]
model_name = info["model_name"]
sim_type   = info["sim_type"]
fmi_ver    = info["fmi_version"]

def field_row(var):
    name  = var["name"]
    val   = var["start"] if var["start"] is not None else 0
    unit  = var["unit"]
    desc  = var["desc"]
    return f"""
    <div class="row">
      <span class="name" title="{desc}">{name}</span>
      <input class="inp" id="p_{name}" type="number" value="{val}" step="any">
      <span class="unit">{unit}</span>
      <span class="desc">{desc}</span>
    </div>"""

input_rows = "".join(field_row(v) for v in info["inputs"])
param_rows = "".join(field_row(v) for v in info["params"])

all_var_names = [v["name"] for v in info["inputs"]] + [v["name"] for v in info["params"]]
collect_js = " + ".join([f'"&{n}=" + document.getElementById("p_{n}").value' for n in all_var_names]) if all_var_names else '""'

widget_html = f"""
<style>
  *{{box-sizing:border-box;margin:0;padding:0}}
  .wrap{{padding:16px;font-family:var(--font-family,system-ui);color:var(--color-text-primary,#111);background:var(--color-background-primary,#fff);max-width:600px}}
  .hdr{{margin-bottom:14px}}
  .title{{font-size:14px;font-weight:700}}
  .sub{{font-size:11px;color:var(--color-text-secondary,#666);margin-top:3px}}
  .section-label{{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.06em;color:var(--color-text-secondary,#888);margin:14px 0 6px}}
  .row{{display:grid;grid-template-columns:130px 90px 40px 1fr;align-items:center;gap:6px;margin-bottom:6px}}
  .name{{font-size:12px;font-weight:500}}
  .inp{{padding:3px 6px;font-size:12px;border:0.5px solid var(--color-border,#ccc);background:var(--color-background-secondary,#f5f5f5);color:var(--color-text-primary,#111);border-radius:3px;width:100%}}
  .unit{{font-size:11px;color:var(--color-text-secondary,#888)}}
  .desc{{font-size:11px;color:var(--color-text-secondary,#888);white-space:nowrap;overflow:hidden;text-overflow:ellipsis}}
  .divider{{border:none;border-top:0.5px solid var(--color-border,#e5e5e5);margin:14px 0}}
  .btn{{width:100%;padding:9px;font-size:13px;font-weight:600;border:0.5px solid var(--color-border,#ccc);background:var(--color-background-secondary,#f5f5f5);color:var(--color-text-primary,#111);cursor:pointer;border-radius:4px;margin-top:14px}}
</style>
<div class="wrap">
  <div class="hdr">
    <div class="title">{model_name}</div>
    <div class="sub">FMI {fmi_ver} · {sim_type}</div>
  </div>

  <div class="section-label">Simulation Time</div>
  <div class="row"><span class="name">Start</span><input class="inp" id="start" type="number" value="{info['start']}" step="any"><span class="unit">s</span><span class="desc"></span></div>
  <div class="row"><span class="name">Stop</span><input class="inp" id="stop" type="number" value="{info['stop']}" step="any"><span class="unit">s</span><span class="desc"></span></div>
  <div class="row"><span class="name">Step size</span><input class="inp" id="step" type="number" value="{info['step']}" step="any"><span class="unit">s</span><span class="desc"></span></div>

  {'<hr class="divider"><div class="section-label">Input Variables</div>' + input_rows if info["inputs"] else ""}
  {'<hr class="divider"><div class="section-label">Tunable Parameters</div>' + param_rows if info["params"] else ""}

  <button class="btn" onclick="runSim()">&#9654; Run Simulation</button>
</div>
<script>
function runSim() {{
  var start = document.getElementById('start').value;
  var stop  = document.getElementById('stop').value;
  var step  = document.getElementById('step').value;
  var extra = {collect_js};
  sendPrompt('SIMULATE:{fmu_path} start=' + start + ' stop=' + stop + ' step=' + step + extra);
}}
</script>
"""

widget_path = "/tmp/" + "{model_name}".replace(" ", "_") + "_config.html"
with open(widget_path, "w") as f:
    f.write(widget_html)
print(widget_path)
```

Call `show_widget` with the widget HTML. Then ask the user to review the parameters and click **▶ Run Simulation** when ready.

---

## Phase 2 — Run simulation and show results

Triggered when `$ARGUMENTS` starts with `SIMULATE:`.

### Step 1 — Parse the message

The format is:
```
SIMULATE:<fmu_path> start=X stop=Y step=Z [param1=V1] [param2=V2] ...
```

Extract:
- `FMU_PATH` — the path after `SIMULATE:`
- `start`, `stop`, `step` — simulation time settings
- All remaining `key=value` pairs → `start_values` dict (parameter overrides)

### Step 2 — Run the simulation

```bash
uv run --with fmpy --with matplotlib python3 - <<'PYEOF'
import sys, os, json, numpy as np, matplotlib, platform as _platform
matplotlib.use('Agg')
import matplotlib.pyplot as plt
from fmpy import read_model_description, supported_platforms
from fmpy.simulation import simulate_fmu

FMU_PATH    = "$FMU_PATH"
start_time  = $start
stop_time   = $stop
step_size   = $step
start_values = $start_values_dict   # e.g. {"g": -5.0, "e": 0.9}
OUTPUT_CSV  = "/tmp/$MODEL_NAME_results.csv"

md = read_model_description(FMU_PATH)
sim_type = "CoSimulation" if md.coSimulation else "ModelExchange"
unit_map = {v.name: (v.unit or "") for v in md.modelVariables}
model_name = md.modelName

print(f"Running {sim_type}: {start_time}s → {stop_time}s, step={step_size}s")
if start_values:
    print(f"Parameter overrides: {start_values}")

result = simulate_fmu(
    FMU_PATH,
    start_time=start_time,
    stop_time=stop_time,
    output_interval=step_size,
    fmi_type=sim_type,
    start_values=start_values if start_values else None,
)

np.savetxt(OUTPUT_CSV, result, delimiter=",",
           header=",".join(result.dtype.names), comments="")
print(f"Saved {len(result)} rows to: {OUTPUT_CSV}")

# ── Static PNG ────────────────────────────────────────────────────────────────
time_col = result.dtype.names[0]
outputs  = [n for n in result.dtype.names if n != time_col]
n = len(outputs)
fig, axes = plt.subplots(n, 1, figsize=(11, 3.2 * n), sharex=True)
if n == 1: axes = [axes]
colors = plt.rcParams['axes.prop_cycle'].by_key()['color']
for i, (ax, col) in enumerate(zip(axes, outputs)):
    unit = unit_map.get(col, ""); label = f"{col} ({unit})" if unit else col
    ax.plot(result[time_col], result[col], color=colors[i % len(colors)], linewidth=1.5)
    ax.set_ylabel(label); ax.grid(True, alpha=0.4)
axes[-1].set_xlabel("Time (s)")
fig.suptitle(f"{model_name}  —  FMI {md.fmiVersion}  {sim_type}", fontsize=13, y=1.01)
plt.tight_layout()
plot_path = OUTPUT_CSV.replace(".csv", ".png")
plt.savefig(plot_path, dpi=150, bbox_inches="tight")
print(f"PNG: {plot_path}")

# ── Summary stats ─────────────────────────────────────────────────────────────
print("\nRESULTS:")
for col in outputs:
    vals = result[col]; unit = unit_map.get(col, "")
    print(f"  {col}: min={vals.min():.4f}  max={vals.max():.4f}  final={vals[-1]:.4f}  {unit}")

# ── Interactive widget ─────────────────────────────────────────────────────────
step_n   = max(1, len(result) // 500)
t_data   = result[time_col][::step_n].tolist()
palette  = ["#4e8ef7","#f76e4e","#4ecf8e","#f7c94e","#a44ef7","#4ef7f0"]
datasets = []
for i, col in enumerate(outputs):
    unit = unit_map.get(col, ""); label = f"{col} ({unit})" if unit else col
    datasets.append({"label": label, "data": result[col][::step_n].tolist(),
        "borderColor": palette[i % len(palette)], "backgroundColor": palette[i % len(palette)],
        "borderWidth": 1.5, "pointRadius": 0, "tension": 0.1})

has_pid = any(v.name in ("Kp","Ki","Kd","Ti","Td") for v in md.modelVariables)
tune_btn = '<button class="btn" onclick="sendPrompt(\'Tune the PID for this FMU\')">Tune PID</button>' if has_pid else ""
adjust_btn = f'<button class="btn-sec" onclick="sendPrompt(\'Adjust parameters for {model_name}\')">Adjust Parameters</button>'

override_text = ""
if start_values:
    override_text = "  · " + "  ".join(f"{k}={v}" for k,v in start_values.items())

widget_html = f"""
<style>
  *{{box-sizing:border-box;margin:0;padding:0}}
  .wrap{{padding:16px;font-family:var(--font-family,system-ui);color:var(--color-text-primary,#111);background:var(--color-background-primary,#fff)}}
  .hdr{{display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:12px}}
  .title{{font-size:13px;font-weight:600}}
  .sub{{font-size:11px;color:var(--color-text-secondary,#666);margin-top:3px}}
  .actions{{display:flex;gap:6px}}
  .btn{{padding:5px 12px;font-size:12px;border:0.5px solid var(--color-border,#ccc);background:var(--color-background-secondary,#f5f5f5);color:var(--color-text-primary,#111);cursor:pointer;border-radius:4px}}
  .btn-sec{{padding:5px 12px;font-size:12px;border:0.5px solid var(--color-border,#ccc);background:transparent;color:var(--color-text-secondary,#666);cursor:pointer;border-radius:4px}}
  canvas{{width:100%!important}}
</style>
<div class="wrap">
  <div class="hdr">
    <div>
      <div class="title">{model_name}</div>
      <div class="sub">FMI {md.fmiVersion} · {sim_type} · {start_time}s → {stop_time}s{override_text}</div>
    </div>
    <div class="actions">{adjust_btn}{tune_btn}</div>
  </div>
  <canvas id="simChart"></canvas>
</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<script>
new Chart(document.getElementById('simChart'),{{
  type:'line',
  data:{{labels:{json.dumps([round(x,4) for x in t_data])},datasets:{json.dumps(datasets)}}},
  options:{{animation:false,responsive:true,interaction:{{mode:'index',intersect:false}},
    plugins:{{legend:{{labels:{{font:{{size:11}},boxWidth:12}}}},tooltip:{{bodyFont:{{size:11}},titleFont:{{size:11}}}}}},
    scales:{{
      x:{{title:{{display:true,text:'Time (s)',font:{{size:11}}}},ticks:{{maxTicksLimit:10,font:{{size:10}}}},grid:{{color:'rgba(128,128,128,0.15)'}}}},
      y:{{ticks:{{font:{{size:10}}}},grid:{{color:'rgba(128,128,128,0.15)'}}}}
    }}
  }}
}});
</script>
"""

widget_path = OUTPUT_CSV.replace(".csv", "_widget.html")
with open(widget_path, "w") as f:
    f.write(widget_html)
print(f"Widget: {widget_path}")
PYEOF
```

### Step 3 — Display results

1. Use the Read tool to show the PNG inline
2. Read the widget HTML file and call `show_widget` with its contents (or `open` it in the browser as fallback)
3. Give a plain-language interpretation of the results:
   - What happened in the simulation?
   - Are the results as expected?
   - Note anything interesting (oscillations, settling, instability, steady-state error)
   - If parameters were overridden, comment on how they affected the outcome

---

## Error handling

| Situation | Response |
|-----------|----------|
| FMU not found | Search with Glob, suggest nearby `.fmu` files |
| Wrong platform | "This FMU was compiled for `<platform>` and can't run on your OS. Ask the model author for a `<os_tag>` build." |
| Simulation crash | Show error, suggest reducing step size |
| `uv` not found | "Install uv: `curl -LsSf https://astral.sh/uv/install.sh \| sh`" |

---

## Usage examples

```
/fmpy-simulate ./models/BouncingBall.fmu
/fmpy-simulate ./models/fopdt_pi.fmu
```
