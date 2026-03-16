# fmi-skills

Claude Code skills and plugins for working with Functional Mock-up Units (FMUs) and the FMI standard.

## Plugins

### `fmpy-simulate`

Simulate any FMU file using [FMPy](https://github.com/CATIA-Systems/FMPy). Supports FMI 1.0, 2.0, and 3.0, both CoSimulation and ModelExchange.

**Features:**
- Auto-detects simulation type and default experiment parameters from the FMU
- Platform compatibility check before attempting simulation
- Saves results to CSV
- Generates a plot of all output variables
- No manual dependency installation — uses `uv run --with fmpy`

**Usage:**
```
/fmpy-simulate ./model.fmu
/fmpy-simulate ./model.fmu stop=10 step=0.001
/fmpy-simulate ./model.fmu start=1 stop=20 out=./results/sim.csv
```

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add Novia-RDI-Seafaring/fmi-skills
```

Install the simulation plugin:

```
/plugin install fmpy-simulate@fmi-skills
```

## Requirements

- [Claude Code](https://claude.ai/code)
- [uv](https://github.com/astral-sh/uv) — `curl -LsSf https://astral.sh/uv/install.sh | sh`

## Citation

If you use these skills in academic work, please cite:

```bibtex
@software{bjorkskog2026fmiskills,
  author       = {Bj{\"o}rkskog, Christoffer, Mikael Manng{\aa}rd},
  title        = {{fmi-skills}: Claude Code Skills for FMU Simulation},
  year         = {2026},
  publisher    = {GitHub},
  organization = {Novia University of Applied Sciences, RDI Seafaring},
  url          = {https://github.com/Novia-RDI-Seafaring/fmi-skills}
}
```

## License

MIT

## Authors

- Christoffer Björkskog — [christoffer.bjorkskog@novia.fi](mailto:christoffer.bjorkskog@novia.fi)
- Mikael Manngård — [mikael.manngard@novia.fi](mailto:mikael.manngard@novia.fi)

Novia University of Applied Sciences, RDI Seafaring
