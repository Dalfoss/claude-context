---
name: rf-pcb-design-process
description: Use this skill to lay out and run an iterative simulation-driven RF / microwave PCB design project on disk. Provides the directory structure, file templates, and git workflow for capturing each iteration's parameters and distilled results reproducibly, and for producing a final postmortem at the end. Trigger when the user (or the rf-pcb-engineer agent) wants to start an RF PCB design project, log a new iteration, record simulation results, or write up the final design. This skill owns the on-disk mechanics; design judgement lives with the agent or the user.
---

# RF PCB Design Process — file layout and conventions

This skill is the **mechanics** of an iterative, simulation-driven RF / microwave PCB design project. It owns the on-disk structure, the file templates, and the git workflow that make iterations reproducible months later. It does **not** own judgement about what to design, when to stop, which tool to use, or how to interpret results — those belong to the agent (`rf-pcb-engineer`) or the user.

The skill assumes **KiCad** for schematic and layout and **OpenEMS** for EM simulation. The simulation entry point depends on which **simulation mode** the iteration uses (see next section).

## How iterations are tracked: one repo, one commit per iteration

The whole project is a **single git repository**, initialized at project start. There is **one working simulation set** (`sim/`) that mutates in place from one iteration to the next — the model and parameters are never duplicated into per-iteration directories. Each iteration is **one commit, tagged `iter-NNN`**. Git history *is* the snapshot mechanism:

- To recover an iteration's exact inputs: `git checkout iter-007`.
- To see what changed between two iterations: `git diff iter-006 iter-007 -- sim/`.

This keeps the filesystem flat and small, and makes the diff between iterations a one-line parameter change instead of two near-identical directory trees.

**What is committed:** the working simulation set (`sim/`), the KiCad project (`kicad/`), the per-iteration markdown records (`iterations/iter-NNN.md`), and the machine-readable trend file (`history.json`). These are all small text and diff cleanly.

**What is gitignored:** `raw/` and `plots/` — the regenerable bulk (touchstone files, field dumps, logs, images). They are reproducible by checking out the iteration tag and re-running the model, so they never enter git history. The *numbers* extracted from a run are preserved in `history.json` and the iteration markdown, which is all that needs to survive.

## Simulation modes

An RF PCB project commonly uses one or both modes. The agent picks the mode per structure; the skill defines what each looks like on disk.

- **Mode A — parametric** (default for antennas and novel-structure design). The OpenEMS model is built in Python from CSXCAD primitives, parameterized by `sim/params.yaml`. KiCad is the *output*: the converged geometry is exported as a KiCad footprint at the end of the loop. Tooling pattern: [`matthuszagh/pyems`](https://github.com/matthuszagh/pyems) or hand-rolled CSXCAD primitives.
- **Mode B — routed-trace verification** (default for SI, impedance, differential-pair work, and verifying matching-network launches on a routed PA board). KiCad is the *input*: the layout is exported as gerbers + drill + stackup, and [`antmicro/gerber2ems`](https://github.com/antmicro/gerber2ems) (typically paired with [`antmicro/kicad-si-simulation-wrapper`](https://github.com/antmicro/kicad-si-simulation-wrapper) to select nets directly from `.kicad_pcb`) builds the OpenEMS simulation. Gerber2ems does **not** model components — capacitors are treated as shorts — so the EM region under simulation is the passive routed structure only. Whole-board simulation is impractical; pick a region of interest.

`sim/params.yaml` records which mode the iteration uses via a `mode:` field. The `sim/` contents differ per mode (below). The `raw/summary.json` → `history.json` contract is identical across both modes, so plots and the postmortem do not need to know which mode produced the numbers.

## Directory layout

One directory per design project — and that directory is one git repo:

```
<project>/                          # one git repo, git init at project start
├── .gitignore                      # ignores raw/ and plots/ (regenerable bulk)
├── goals.md                        # the contract — written first
├── stackup.md                      # layer stackup, materials, vendor
├── tooling.md                      # tool versions + calibration record
├── references/                     # datasheets, app notes, prior-art measurements
├── kicad/                          # KiCad project — Mode B: input; Mode A: receives final geometry
├── sim/                            # SINGLE working simulation set — mutates in place, committed
│   ├── model.py                    # OpenEMS driver — Mode A: parametric CSXCAD; Mode B: gerber2ems wrapper
│   ├── params.yaml                 # parameters defining the CURRENT iteration
│   ├── mesh.yaml                   # mesh resolution (Mode A) / gerber2ems config (Mode B)
│   ├── ports.yaml                  # port definitions, excitation
│   └── gerber/                     # Mode B only: gerbers + drill + stackup snapshot fed to gerber2ems
├── raw/                            # GITIGNORED — touchstone, field dumps, log, summary.json (regenerable)
├── plots/                          # GITIGNORED — S-param plots, smith charts, field images (regenerable)
├── iterations/
│   ├── iter-001.md                 # one file per iteration: hypothesis, param-diff, results, notes
│   └── iter-NNN.md
├── history.json                    # machine-readable per-iteration summaries — the trend-plot source
└── final/
    ├── summary.md                  # postmortem: journey + final + open risks
    └── manufacturing/              # DRC report, fab notes, panelization, exported gerbers/STEP/BOM
```

The final design source is `kicad/` at the `final` tag; `final/manufacturing/` holds the exported fabrication outputs and reports.

## Git workflow per iteration

1. Edit `sim/params.yaml` (and `sim/model.py` only if the structure itself changed). One hypothesis per iteration.
2. Run `sim/model.py`. It writes `raw/` outputs including `raw/summary.json`, and `plots/`.
3. Append `raw/summary.json` into `history.json` under the new iteration number (schema below), and write `iterations/iter-NNN.md`.
4. Commit and tag:
   ```
   git add sim/ kicad/ iterations/iter-NNN.md history.json
   git commit -m "iter-NNN(<project>): <hypothesis one-liner>"
   git tag iter-NNN
   ```
   The commit message **identifies both the iteration and the project**. The tag is the recovery handle — the iteration markdown does not store a SHA.

`parent` (logical lineage) is recorded in the iteration markdown and may differ from the git parent: if iteration 7 logically branches from iteration 3 rather than 6, the git history is still chronological but `parent: 3` records the design lineage. Keep history linear; the `parent` field is the source of truth for lineage.

## Templates

Each template below is what should appear in the file when first created. Tables and prose; no decorative formatting. These files outlive the project.

### `.gitignore`

````gitignore
# Regenerable simulation bulk — recreate by `git checkout iter-NNN` and re-running sim/model.py.
# The numbers that matter survive in history.json and the iteration markdown.
raw/
plots/
````

### `goals.md`

````markdown
# <project> — design goals & constraints

## Functional
<one paragraph: what this board does, where it sits in the system>

## Acceptance criteria
Every criterion has units and a threshold. Each is tagged with its verification path:
- **[sim]** verifiable in OpenEMS in this project
- **[bench]** only verifiable on hardware
- **[ext]** needs a tool outside this flow (SPICE, thermal FEM, ADS, VNA, etc.)

- [ ] **[sim]** S11 < −10 dB across 2.40–2.48 GHz
- [ ] **[sim]** Insertion loss ≤ 0.5 dB across passband  (positive-loss convention: a measured `loss_db = 0.32` passes; lower is better)
- [ ] **[sim]** Unconditional stability **μ-factor > 1** DC → f_max (cascaded with device S2P at the operating bias point). Rollett `k > 1, |Δ| < 1` as secondary cross-check, restricted to `|S21| > −20 dB` frequencies — see "Sign conventions" below for why.
- [ ] **[bench]** EVM < 3 % at +20 dBm output
- [ ] **[ext]** Junction temperature < 110 °C at max output (thermal FEM)

## Mechanical / form factor
- Outline: <mm × mm>
- Mounting: <hole pattern, connector positions>
- Height budget: <mm including connectors>

## Regulatory / EMC
- Target: <FCC Part 15 B, CE EN 55032 Class A/B, MIL-STD-461, etc.>
- Required margin: <dB below limit>

## BOM / cost
- Target unit cost at qty <N>: <€/$>
- Forbidden parts or vendors: <list>
- Preferred component series / families: <list>

## Manufacturing
- Fab: <vendor or "TBD — assumed JLC/PCBWay class">
- Min trace / space: <mil>
- Layer count cap: <N>
- Controlled impedance: <yes/no, tolerance ±%>
- Surface finish: <ENIG / HASL / ...>

## Out of scope
<things explicitly NOT goals — this section is the scope-creep firewall>
````

### `stackup.md`

````markdown
# Stackup

| Layer    | Material | Thickness | Cu weight | Dk  | Df    | Notes                |
|----------|----------|-----------|-----------|-----|-------|----------------------|
| L1 top   | Cu       | 35 µm     | 1 oz      | —   | —     | signal + RF          |
| Core     | FR4      | 0.51 mm   | —         | 4.3 | 0.02  | nominal at 1 GHz     |
| L2 gnd   | Cu       | 35 µm     | 1 oz      | —   | —     | continuous reference |
| Prepreg  | FR4      | 0.20 mm   | —         | 4.3 | 0.02  |                      |
| L3 pwr   | Cu       | 35 µm     | 1 oz      | —   | —     |                      |
| Core     | FR4      | 0.51 mm   | —         | 4.3 | 0.02  |                      |
| L4 bot   | Cu       | 35 µm     | 1 oz      | —   | —     |                      |

**Vendor / stack name**: <e.g. JLCPCB JLC04161H-7628>
**Dk uncertainty**: ±0.2 across batches — record the value used in each iter's `sim/params.yaml`
**Controlled-impedance reference**: <50 Ω microstrip L1-ref-L2 → width 0.33 mm>
````

### `tooling.md`

````markdown
# Tooling

- KiCad: <version>
- OpenEMS: <version>
- CSXCAD: <version>
- Python: <version>
- gerber2ems: <version or "not used">             # Mode B bridge
- kicad-si-simulation-wrapper: <version or "not used">
- pyems: <version or "not used">                  # Mode A helper, if used
- OS / CPU: <for runtime context>

## Calibration
Reference structure used to validate the simulation setup against a known answer:
- Structure: <e.g. 50 Ω through-line, length 25 mm, on the project stackup>
- Expected: |S21| > −0.3 dB at 2.4 GHz, group delay flat to ±10 ps
- Measured in sim (iteration or referenced run): <numbers>
- Agreement: <pass / fail, notes>

Re-run when: OpenEMS / CSXCAD version changes, mesh strategy changes, stackup changes.
````

### `sim/params.yaml`

The current iteration's parameters. This file mutates in place each iteration; its full state at iteration N is recoverable with `git checkout iter-NNN`.

````yaml
iteration: 7
date: 2026-05-26
parent: 6                            # logical lineage; null only for iter-001
mode: parametric                     # "parametric" (Mode A) or "gerber2ems" (Mode B)
hypothesis: >
  Shortening the antenna arm by 1.5 mm should move resonance from 2.38 GHz
  up to ~2.45 GHz without significantly affecting bandwidth.

changes_from_parent:
  - antenna.arm_length_mm: 32.0 → 30.5
  # everything else inherited from parent

sweep: null                          # or e.g. {var: antenna.arm_length_mm, values: [29.5, 30.0, 30.5, 31.0]}

geometry:
  substrate:
    material: FR4
    er: 4.3
    tan_d: 0.02
    height_mm: 1.6
  copper:
    weight_oz: 1
    thickness_um: 35
  antenna:
    type: inverted-F
    arm_length_mm: 30.5
    feed_offset_mm: 5.0
    ground_clearance_mm: 8.0
    # ... whatever else is being varied

simulation_targets:                  # which acceptance criteria THIS run measures
  - s11_band_min_db: { range_ghz: [2.40, 2.48], threshold: -10 }
  - gain_at_f0_dbi:  { f_ghz: 2.45, threshold: 0 }
````

### `sim/model.py`

OpenEMS driver. The driver's *internals* differ by mode; its *interface* (inputs and outputs) does not.

**Common contract — both modes:**
- Reads `params.yaml`, `mesh.yaml`, `ports.yaml` from its own directory. No hardcoded geometry, frequencies, or thresholds — lift them into yaml.
- Deterministic for a given input + tool versions. Any random seed is recorded in `raw/summary.json`.
- Writes (all into the gitignored `raw/` and `plots/`):
  - raw touchstone (`raw/<port-pair>.sNp`)
  - field dumps if requested (`raw/E_field_*.h5`, `raw/H_field_*.h5`)
  - one `raw/summary.json` keyed by acceptance-criterion name (schema below).
- The iteration-logging step then appends `raw/summary.json` into `history.json` under the iteration number; `raw/` itself is never committed.

**Mode A — parametric (CSXCAD primitives):**
- Build the EM-critical structure directly in Python from `params.yaml.geometry`. Use `matthuszagh/pyems` patterns or hand-rolled CSXCAD calls (`AddBox`, `AddPolygon`, `AddMaterial`, etc.).
- Mesh refinement regions are declared by *name* in `mesh.yaml` (e.g. `antenna_feed`, `via_fence`); the driver derives their coordinates from current geometry so a geometry change automatically moves the refinement.
- Iteration cost: a yaml edit.

**Mode B — gerber2ems:**
- Snapshot the gerbers + drill + stackup that come out of `kicad-cli pcb export gerbers / drill` into `sim/gerber/`. These are the simulation input and are committed — never modify them by hand.
- The driver calls `gerber2ems` with the project's stackup and the net selection (typically generated via `antmicro/kicad-si-simulation-wrapper` from `.kicad_pcb` and recorded alongside in `sim/nets.yaml`).
- `mesh.yaml` in this mode holds gerber2ems-specific config (adaptive grid settings, ROI bounding box) rather than CSXCAD mesh primitives.
- Iteration cost: a KiCad layout edit + gerber re-export. Use Mode B sparingly — Mode A iterates faster.

**`raw/summary.json` schema (both modes) — the per-run output:**

````json
{
  "s11_band_max_db":       { "value": -14.2, "unit": "dB",  "pass": true,  "target": -10 },
  "input_loss_at_f0_db":   { "value":  0.18, "unit": "dB",  "pass": true,  "target":   0.5 },
  "stability_mu_min":      { "value":  2.30, "unit": "",    "pass": true,  "target":   1.0 }
}
````

**Required casts before `json.dump`:** Python 3.13's json module rejects
`numpy.bool_` and `numpy.float64`. Always wrap your `pass` and `value`
fields with explicit Python casts before writing:
```python
"pass":  bool(measured < threshold),    # NOT just `measured < threshold`
"value": float(measured),               # NOT just `measured`
```
Symptom if you forget: `TypeError: Object of type bool is not JSON
serializable` at the very end of an otherwise-successful run, after
hours of simulation. Cast at the boundary.

### `sim/mesh.yaml`

````yaml
units: mm
base_resolution:
  dx: 0.5
  dy: 0.5
  dz: 0.1                            # finer in the stackup direction
refinement:
  - region: antenna_feed
    resolution: 0.1
  - region: via_fence
    resolution: 0.2
boundary_conditions:
  x: [PML_8, PML_8]
  y: [PML_8, PML_8]
  z: [PML_8, PML_8]
````

### `sim/ports.yaml`

````yaml
ports:
  - id: 1
    type: lumped
    location: antenna_feed
    impedance_ohm: 50
    excitation: gaussian
    f_center_ghz: 2.45
    f_bandwidth_ghz: 1.0
````

### `history.json`

The machine-readable contract and the source for any trend plot across the project. One entry appended per iteration; the per-criterion object is exactly the `raw/summary.json` schema, nested under `iterations[]`. Keep criterion keys identical across iterations so a small script can plot any criterion over the whole history without parsing markdown.

````json
{
  "project": "<project>",
  "iterations": [
    {
      "iteration": 7,
      "parent": 6,
      "tag": "iter-007",
      "date": "2026-05-26",
      "mode": "parametric",
      "criteria": {
        "s11_band_max_db":     { "value": -14.2, "unit": "dB", "pass": true, "target": -10 },
        "input_loss_at_f0_db": { "value":  0.18, "unit": "dB", "pass": true, "target":  0.5 },
        "stability_mu_min":    { "value":  2.30, "unit": "",   "pass": true, "target":  1.0 }
      }
    }
  ]
}
````

### `iterations/iter-NNN.md`

One file per iteration. Folds the old `params`-diff, `results`, and `notes` into a single record. The full geometry lives in `sim/params.yaml` at tag `iter-NNN`; this file records only the *diff from parent* plus the distilled results and narrative.

````markdown
# Iteration <N> — <one-line title>

- date: <YYYY-MM-DD>
- parent: <N-1 or logical parent; null only for iter-001>
- mode: <parametric | gerber2ems>
- recover inputs: `git checkout iter-NNN`

## Hypothesis
<from sim/params.yaml — restated for readability>

## Changes from parent
- antenna.arm_length_mm: 32.0 → 30.5
- # full geometry is in sim/params.yaml at tag iter-NNN

## Sweep
<null, or {var: antenna.arm_length_mm, values: [29.5, 30.0, 30.5, 31.0]}>

## Results — [sim]-verifiable subset
Sign convention: see "Sign conventions" in the skill. Insertion-loss-style
criteria use a positive `loss_db` value with pass `< threshold`; return-loss-
and S-parameter-style criteria use natural negative `_db` values with pass
`< threshold` (deeper = better).

| Criterion             | Target          | Measured           | Pass? |
|-----------------------|-----------------|--------------------|-------|
| s11_band_max_db       | < −10 dB        | −14.2 dB (worst)   | ✅    |
| input_loss_at_f0_db   | < 0.5 dB        | 0.18 dB            | ✅    |
| stability_mu_min      | > 1.0           | +2.3 (in band)     | ✅    |
| gain_at_f0_dbi        | > 0 dBi         | −1.1 dBi           | ❌    |

## Diff vs iter-<parent>
For each criterion in both iterations, the numeric movement. Generated by
post-processing from `history.json`; omit this section in iter-001.

| Criterion             | iter-<parent>   | iter-<N>          | Δ          |
|-----------------------|-----------------|-------------------|------------|
| s11_band_max_db       | −12.3 dB        | −14.2 dB          | −1.9 dB ✓  |
| input_loss_at_f0_db   | 0.21 dB         | 0.18 dB           | −0.03 dB ✓ |
| stability_mu_min      | +1.8            | +2.3              | +0.5 ✓     |

## Key derived numbers
- Resonant frequency: <GHz>
- −10 dB bandwidth: <MHz>
- Input impedance at f₀: <Ω> (real, imag)
- Radiation efficiency: <%>
- Peak gain direction: <θ, φ>

## What actually happened
<observation, citing specific plots in plots/ — regenerate with `git checkout iter-NNN` if needed>

## Why — physical interpretation
<not just "S11 went down" but "shortening the arm raised resonance because the
electrical length decreased; gain dropped because the new ground clearance
reduced the effective aperture">

## What this means for the next iteration
<one sentence — becomes the next iter's hypothesis>

## Misc observations worth not forgetting
<e.g. "mesh was borderline at the feed gap; refine further if dB-level
agreement is needed in subsequent iters">

## Verdict
<pass | fail — list which criteria failed if any>
````

### Sign conventions (read once, then forget)

To avoid the "is −0.07 better than −0.3?" confusion that recurs across
iterations, criteria use these conventions:

- **Insertion loss**: report as a **positive `*_loss_db` value**. Pass test
  is `value < threshold` (lower is better). Example:
  `input_launch_loss_db: 0.18  →  pass if < 0.5`.
- **Return loss / S-parameters**: report as **natural-sign `*_db` value**
  (negative for passive structures). Pass test is `value < threshold`
  (more negative = better match). Example:
  `s11_band_max_db: −14.2  →  pass if < −10`.
- **Stability**: report as μ-factor and (optionally) Rollett k, both
  dimensionless. Pass test is `value > threshold` (typically 1.0). The
  **μ-factor is the primary stability metric** because it's numerically
  robust where `|S12·S21| → 0`. Rollett k may be reported as a secondary
  cross-check, **restricted to frequencies where `|S21| > −20 dB`** —
  outside that range the k-formula's denominator blows up and produces
  spurious huge-magnitude "instabilities" that are not real.
- **Gain / efficiency**: report as positive dB / % / dBi. Pass test is
  `value > threshold` (higher is better).

The `pass` direction is encoded by the metric *name* — `*_loss_db`,
`*_db`, `*_min`, `*_max`, `*_dbi`, `*_pct`. If a new criterion doesn't
fit one of those patterns, add it to this section before introducing it.

### `final/summary.md`

````markdown
# <project> — final design summary

## Final design
- Stackup: see `stackup.md`
- Key geometry: <table or short prose>
- Design source: `kicad/` at tag `final`
- Gerbers + BOM + STEP: `final/manufacturing/`

## Acceptance criteria — status
| Criterion        | Target | Final value         | Iter achieved | Verification |
|------------------|--------|---------------------|----------------|--------------|
| <criterion>      | <...>  | <...>               | iter-007       | sim          |
| <criterion>      | <...>  | pending             | —              | bench        |
| <criterion>      | <...>  | pending             | —              | ext          |

## Design journey

### What worked
- **<change>** (iter-003 → iter-004): <one sentence + evidence>

### What didn't work
- **<change>** (iter-005): <one sentence on why it failed — saves the next engineer a re-walk>

### Dead ends explored
- **<approach>**: <considered and rejected, with the iteration that disproved it>

## Open risks (simulation did not cover these)
- <e.g. dielectric variation across FR4 batches>
- <e.g. connector launch parasitics not modeled>
- <e.g. assembly tolerance on critical dimension>
- <e.g. thermal effect on bias point — needs FEM>

## Recommended next steps
- Bench validation: <what to measure first, on what equipment>
- DFM review with fab
- Thermal simulation if PA / high-current
- Compliance pre-scan if EMC is in scope
````

## Conventions and rules

1. **Past iterations are immutable — git enforces it.** Each iteration is a commit tagged `iter-NNN`; never rewrite history (no `--amend`, no rebase across a published iteration). If a bug is found in iter-003's sim, create iter-004 that re-runs it correctly and explain in iter-004's markdown. Editing history breaks the story the postmortem will tell.
2. **One git repo per project, initialized at project start.** `git init` the project directory and commit the scaffold (`goals.md`, `stackup.md`, `tooling.md`, `.gitignore`) before iter-001. The whole project — `sim/`, `kicad/`, iteration records, `history.json` — is tracked together; the KiCad project lives in the same repo, so each iteration's commit captures the design and the simulation together. No file copying for snapshots; git is the snapshot mechanism. (If KiCad must live in a separate repo for an external reason, record its SHA in the iteration markdown — but the default is one repo.)
3. **`raw/` and `plots/` are gitignored.** They are regenerable from `sim/` at any iteration tag. The numbers that must survive live in `history.json` and the iteration markdown. Do not commit field dumps or images.
4. **Each iteration is one commit, tagged `iter-NNN`, with a project-and-iteration commit message.** Format: `iter-NNN(<project>): <hypothesis one-liner>`. The tag is the recovery handle; the markdown stores no SHA.
5. **Zero-pad iteration numbers** (`iter-001`, not `iter-1`) in both file names and tags so sort matches chronological order. Three digits supports up to 999; if you hit that you probably need a structural rethink instead.
6. **`parent` is mandatory for iter ≥ 002.** Designs branch; the parent chain (recorded in the markdown and `history.json`) is how lineage is reconstructed, independent of git's chronological parent. Only iter-001 has `parent: null`.
7. **`history.json` is the machine-readable contract.** Append one entry per iteration. Keep the criterion keys identical across iterations of the same project so a small script can plot any criterion over the iteration history without parsing markdown.
8. **`final/` does not exist until all `[sim]` criteria pass**, at which point the design source is tagged `final`. If tempted to create it early because "we're close enough", you aren't — keep iterating or escalate the criterion as infeasible.
9. **Record tool versions in `tooling.md`** and re-run the calibration reference if any version changes mid-project. Without this, "the simulation used to give X" becomes unfalsifiable.
10. **Parametric simulation models, not hardcoded geometry.** A `model.py` that rebuilds from `params.yaml` makes the next iteration a yaml edit; hardcoded coordinates make it a rewrite. Non-parametric models are allowed when geometry resists parameterization, but call them out in the iteration markdown.
11. **Parameter sweeps live inside a single iteration**, not as separate iterations. Use the `sweep` field in `sim/params.yaml` and produce one summary per swept point under `raw/sweep/<value>/` (gitignored). Record the swept results in the iteration markdown and the converged/representative point in `history.json`. One hypothesis, one iteration file, one verdict — that's what keeps the iteration log a readable narrative.
12. **No decorative formatting in templates.** Tables and plain prose. These files are read by future engineers (and future LLM context windows); make them scannable.
13. **Cast numpy scalars to Python types before `json.dump`.** Python 3.13's json rejects `numpy.bool_` and `numpy.float64`. Always wrap with `bool(...)` / `float(...)` at the JSON boundary. Symptom of forgetting: `TypeError: Object of type bool is not JSON serializable` at the very end of an otherwise-successful run.
14. **Each iteration markdown includes a "Diff vs iter-<parent>" table** when the parent exists, generated from `history.json`. Skip the section in iter-001 (no parent). Makes the iteration log readable months later.
15. **Use the μ-factor (Edwards–Sinsky) as the primary stability metric.** Rollett k is a secondary cross-check restricted to frequencies where `|S21| > −20 dB`. See "Sign conventions" above.

## First actions when invoked

1. **New project:** if no `goals.md` exists, write one with the user before any layout. Push back on non-measurable criteria. Refuse to start the loop until every criterion is tagged `[sim] / [bench] / [ext]`. Then `git init` the project and commit the scaffold (`goals.md`, `stackup.md`, `tooling.md`, `.gitignore`).
2. **Existing project:** if `goals.md` exists, check the current `sim/params.yaml` is consistent with it and that the criteria-tagging is current. `git tag` lists the iterations on disk; `history.json` is the numeric history.
3. **"Start a new iteration":** the working set in `sim/` is already the parent's state (it mutates in place). Edit `sim/params.yaml` — bump `iteration`, set `parent`, write the hypothesis and `changes_from_parent`, apply the geometry change — then **stop before running** so the user (or agent) can confirm the hypothesis is worth running. Do not commit yet; uncommitted changes are trivially reverted if the hypothesis is rejected. After the run, log the results, append to `history.json`, write `iterations/iter-NNN.md`, then commit and `git tag iter-NNN`.
4. **"Present the design":** read every `iterations/*.md` and `history.json` and synthesize `final/summary.md` from what's recorded. Tag the design source `final`. Do not invent narrative — use the iteration log as the source of truth.
