---
name: rf-pcb-design-process
description: Use this skill to lay out and run an iterative simulation-driven RF / microwave PCB design project on disk. Provides the directory structure, file templates, and git workflow for capturing each iteration's parameters and distilled results reproducibly, and for producing a final postmortem at the end. Trigger when the user (or the rf-pcb-engineer agent) wants to start an RF PCB design project, log a new iteration, record simulation results, or write up the final design. This skill owns the on-disk mechanics; design judgement lives with the agent or the user.
---

# RF PCB Design Process — file layout and conventions

This skill is the **mechanics** of an iterative, simulation-driven RF / microwave PCB design project. It owns the on-disk structure, the file templates, and the git workflow that make iterations reproducible months later. It does **not** own judgement about what to design, when to stop, which tool to use, or how to interpret results — those belong to the agent (`rf-pcb-engineer`) or the user.

The skill assumes **KiCad** for schematic and layout and **OpenEMS** for EM simulation. The simulation entry point depends on which **simulation mode** the iteration uses (see next section).

## How iterations are tracked: workspace › board › structure › iteration

The repository is a **PCB-design workspace** — kept separate from any software/firmware repo so its iteration tags do not collide with software release tags. One workspace holds **one subdirectory per board**; each board contains one or more independent EM **structures** (the patch element, an array, each filter, the IF termination …).

Each structure has its own working simulation set (`<board>/structures/<name>/sim/`) that mutates in place from one iteration to the next, its own iteration records, and its own `history.json` — their geometries and acceptance criteria are unrelated and they converge independently. Within a structure the model and parameters are never duplicated into per-iteration directories.

The **shared toolchain** — the Python `.venv/` and the built `openEMS/` — lives once at the workspace root (both gitignored). Everything board-specific (`goals.md`, `stackup.md`, `tooling.md`, `kicad/`, `final/`, `structures/`) lives inside that board's subdirectory, because each board has its own substrate and acceptance criteria.

Each iteration is **one commit, tagged `<board>/<structure>/iter-NNN`** — namespaced by board and structure so the same `iter-NNN` number is reusable across structures and boards in the one workspace (git tags permit `/`). Git history *is* the snapshot mechanism:

- Recover an iteration's exact inputs: `git checkout 10-ghz-rf-board/patch-element/iter-007`.
- List a structure's iterations: `git tag --list '10-ghz-rf-board/patch-element/*'`.
- Diff two iterations: `git diff 10-ghz-rf-board/patch-element/iter-006 10-ghz-rf-board/patch-element/iter-007 -- 10-ghz-rf-board/structures/patch-element/sim/`.

This keeps the filesystem flat and small (one mutating `sim/` per structure), and makes the diff between iterations a one-line parameter change.

**What is committed:** each structure's `sim/`, the board's KiCad project (`<board>/kicad/`), the per-iteration markdown records (`<board>/structures/<name>/iterations/iter-NNN.md`), and the per-structure machine-readable trend file (`<board>/structures/<name>/history.json`). These are all small text and diff cleanly.

**What is gitignored:** the workspace-root `.venv/` and `openEMS/` (regenerable per README), plus every `raw/` and `plots/` (the regenerable simulation bulk — touchstone files, field dumps, logs, images). They are reproducible by checking out the iteration tag and re-running the model, so they never enter git history. The *numbers* extracted from a run are preserved in `history.json` and the iteration markdown, which is all that needs to survive.

## Simulation modes

An RF PCB project commonly uses one or both modes. The agent picks the mode per structure; the skill defines what each looks like on disk.

- **Mode A — parametric** (default for antennas and novel-structure design). The OpenEMS model is built in Python from CSXCAD primitives, parameterized by `sim/params.yaml`. KiCad is the *output*: the converged geometry is exported as a KiCad footprint at the end of the loop. Tooling pattern: [`matthuszagh/pyems`](https://github.com/matthuszagh/pyems) or hand-rolled CSXCAD primitives.
- **Mode B — routed-trace verification** (default for SI, impedance, differential-pair work, and verifying matching-network launches on a routed PA board). KiCad is the *input*: the layout is exported as gerbers + drill + stackup, and [`antmicro/gerber2ems`](https://github.com/antmicro/gerber2ems) (typically paired with [`antmicro/kicad-si-simulation-wrapper`](https://github.com/antmicro/kicad-si-simulation-wrapper) to select nets directly from `.kicad_pcb`) builds the OpenEMS simulation. Gerber2ems does **not** model components — capacitors are treated as shorts — so the EM region under simulation is the passive routed structure only. Whole-board simulation is impractical; pick a region of interest.

`sim/params.yaml` records which mode the iteration uses via a `mode:` field. The `sim/` contents differ per mode (below). The `raw/summary.json` → `history.json` contract is identical across both modes, so plots and the postmortem do not need to know which mode produced the numbers.

## Directory layout

One git repo = a **PCB-design workspace** (kept separate from software repos). Shared tooling sits at the workspace root; each **board** is a subdirectory; each EM **structure** is a self-contained design loop under that board's `structures/`:

```
<workspace>/                          # one git repo, git init at start; NOT a subtree of a software repo
├── .gitignore                        # ignores .venv/, openEMS/, raw/, plots/, models/ (regenerable bulk)
├── README.md                         # how to set up the shared env (venv + openEMS build)
├── view-model                        # board-agnostic launcher: open any iteration's CSX XML in AppCSXCAD (template below)
├── .venv/                            # GITIGNORED — shared Python env for all boards
├── openEMS/                          # GITIGNORED — shared built OpenEMS (libs + bindings)
├── <board>/                          # one subdirectory per board (e.g. 10-ghz-rf-board)
│   ├── goals.md                      # BOARD-level: function, signal chain, board-wide constraints + structure index
│   ├── stackup.md                    # this board's substrate/stackup — shared by its structures
│   ├── tooling.md                    # tool versions + calibration record (this board's stackup)
│   ├── references/                   # datasheets, app notes, prior-art measurements
│   ├── kicad/                        # KiCad project for this board; structures land here as footprints
│   ├── structures/
│   │   ├── <structure>/              # one self-contained design loop (e.g. patch-element, prelna-bpf)
│   │   │   ├── goals.md              # THIS structure's acceptance criteria (units + thresholds)
│   │   │   ├── sim/                  # SINGLE working simulation set for this structure — mutates in place
│   │   │   │   ├── model.py          # OpenEMS driver — Mode A: parametric CSXCAD; Mode B: gerber2ems wrapper
│   │   │   │   ├── params.yaml       # parameters defining the CURRENT iteration
│   │   │   │   ├── mesh.yaml         # mesh resolution (Mode A) / gerber2ems config (Mode B)
│   │   │   │   ├── ports.yaml        # port definitions, excitation
│   │   │   │   └── gerber/           # Mode B only: gerbers + drill + stackup snapshot fed to gerber2ems
│   │   │   ├── raw/                  # GITIGNORED — touchstone, field dumps, log, summary.json
│   │   │   ├── plots/                # GITIGNORED — one subdir per run (plots/iter-NNN/) so each iteration's plots survive; sweep points use plots/iter-NNN-<label>/
│   │   │   ├── models/               # GITIGNORED — one subdir per run (models/iter-NNN/): the CSX XML (geometry+mesh+ports) for AppCSXCAD inspection via view-model
│   │   │   ├── iterations/
│   │   │   │   ├── iter-001.md       # one file per iteration: hypothesis, param-diff, results, notes
│   │   │   │   └── iter-NNN.md
│   │   │   └── history.json          # this structure's per-iteration summaries — its trend-plot source
│   │   └── <structure-2>/ …
│   └── final/
│       ├── summary.md                # board postmortem: per-structure journeys + final + open risks
│       └── manufacturing/            # DRC report, fab notes, panelization, exported gerbers/STEP/BOM
└── <board-2>/ …                      # additional boards share the workspace .venv/ + openEMS/
```

The `.venv/` and `openEMS/` are built once per README and shared by every board. `stackup.md` and `tooling.md` live in each board's subdirectory — every structure on that board shares the one substrate and calibrated solver setup. The final design source is `<board>/kicad/` at the `final` tag; `<board>/final/manufacturing/` holds the exported fabrication outputs and reports.

## The stackup is shared, board-level, and mutable

`stackup.md` is not a fixed precondition — the stackup (substrate, thickness, copper, finish) is itself an engineering decision and may change as evidence accumulates (e.g. a thin core the antenna cannot match across band — see the rf-pcb-engineer agent's substrate-thickness guidance). What is immutable is git history, not the stackup choice.

Three rules keep a mutable stackup coherent across the per-structure loops:

1. **Changing the stackup is a board-level event, not a single structure's iteration.** The one substrate is shared by every structure, so a change invalidates every structure's prior `[sim]` passes (they were proven on the old stackup) and forces each to re-verify. Update `stackup.md` and its decision log, re-run the calibration reference (`tooling.md`), and re-open the affected structures. The change and its rationale are recorded in `stackup.md`, not buried in one structure's iteration log.
2. **The stackup window is bounded by every consumer, not just the EM structures.** `stackup.md` carries a **constraint register** (template below): one row per consumer of the shared stackup, each declaring a hard floor, a hard ceiling, a soft preference, and the reason. The consumers include the EM structures (antenna, filters, 50 Ω lines) **and the things OpenEMS cannot simulate but that still bound the laminate** — the packages of the already-chosen active parts (QFN/BGA paddle-via inductance, thermal path, RF-pin launch geometry), assembly, and fab capability. The register is filled at Phase 0, before any structure loop optimises within the window, and the feasible window is the intersection of all rows. If the window is empty, that is a Phase-0 finding that forces a topology/layer-count decision, not something to discover after a structure has spent iterations.
3. **Exploring a candidate stackup is done inside the most-sensitive structure — but "sensitive" is not "authoritative."** Run the comparison as ordinary iterations of whichever structure the change most *benefits* (usually the antenna or the tightest filter), varying the substrate fields in that structure's `sim/params.yaml`. That structure is the most sensitive to the change; it is **not** necessarily the structure that *bounds* the parameter (e.g. the antenna wants a thick core for bandwidth, but the QFN active parts cap thickness — the binding constraint lives outside any EM structure). A stackup-exploration iteration must check its proposed value against the rule-2 constraint register and is invalid if it leaves the feasible window. `params.yaml` is authoritative for a given run; it must agree with `stackup.md` except in these explicit stackup-exploration iterations. When a candidate inside the window wins, adopt it board-wide via rule 1.

Often the criterion that triggers a stackup change is already written into a structure's `goals.md` (e.g. "S11 < −10 dB across the band" on a thin core), so the stackup decision falls out of the normal pass/fail loop rather than being a separate process.

## Git workflow per iteration

Paths below are relative to the board directory `<board>/`.

1. Edit `structures/<structure>/sim/params.yaml` (and `model.py` only if the structure itself changed). One hypothesis per iteration.
2. Run the structure's `sim/model.py`. It writes `raw/` outputs including `raw/summary.json`, and `plots/`.
3. Append `raw/summary.json` into that structure's `history.json` under the new iteration number (schema below), and write `structures/<structure>/iterations/iter-NNN.md`.
4. Commit and tag (tag namespaced by board and structure):
   ```
   git add <board>/structures/<structure>/sim/ <board>/structures/<structure>/iterations/iter-NNN.md <board>/structures/<structure>/history.json <board>/kicad/
   git commit -m "iter-NNN(<board>/<structure>): <hypothesis one-liner>"
   git tag <board>/<structure>/iter-NNN
   ```
   The commit message **identifies the iteration, the board, and the structure**. The tag `<board>/<structure>/iter-NNN` is the recovery handle — the iteration markdown does not store a SHA.

`parent` (logical lineage) is recorded in the iteration markdown and may differ from the git parent: if iteration 7 logically branches from iteration 3 rather than 6, the git history is still chronological but `parent: 3` records the design lineage. Keep history linear; the `parent` field is the source of truth for lineage.

## Templates

Each template below is what should appear in the file when first created. Tables and prose; no decorative formatting. These files outlive the project.

### `.gitignore`

````gitignore
# Shared toolchain — rebuild per README (workspace root)
.venv/
openEMS/

# Regenerable simulation bulk — recreate by `git checkout <board>/<structure>/iter-NNN` and re-running sim/model.py.
# The numbers that matter survive in history.json and the iteration markdown.
raw/
plots/
models/
````

### `view-model` (workspace-root scaffold)

A board-agnostic launcher at the workspace root that opens **one specific iteration's**
saved CSX XML in **AppCSXCAD** (ships in `openEMS/bin/`) — the 3-D viewer for geometry +
mesh + ports. Created once per workspace alongside `README.md`; committed (it is a tiny
helper, not regenerable bulk). It sets `LD_LIBRARY_PATH` to the shared `openEMS/lib` and
takes either the iteration's XML file or its `models/iter-NNN/` directory (which holds
that one XML) — no searching, no "newest" guessing. `chmod +x view-model` after creating.

````bash
#!/usr/bin/env bash
# view-model — open a specific OpenEMS model in AppCSXCAD (geometry + mesh + ports).
# Point it at one iteration: the iteration's CSX XML, or its models/iter-NNN/ dir.
#   ./view-model path/to/structures/<name>/models/iter-006-H
#   ./view-model path/to/structures/<name>/models/iter-006-H/array.xml
# VIEW_DRYRUN=1 prints the resolved path without opening the GUI.
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"          # workspace root
APP="${APPCSXCAD:-$ROOT/openEMS/bin/AppCSXCAD}"
export LD_LIBRARY_PATH="$ROOT/openEMS/lib:${LD_LIBRARY_PATH:-}"
arg="${1:-}"
[ -z "$arg" ] && { echo "usage: view-model <iteration-dir | model.xml>" >&2; exit 1; }
if [ -f "$arg" ]; then xml="$arg"
elif [ -d "$arg" ]; then
    shopt -s nullglob; xmls=("$arg"/*.xml)
    if [ "${#xmls[@]}" -eq 0 ]; then echo "view-model: no .xml in $arg" >&2; exit 1
    elif [ "${#xmls[@]}" -gt 1 ]; then echo "view-model: multiple .xml in $arg — point at one:" >&2; printf '  %s\n' "${xmls[@]}" >&2; exit 1; fi
    xml="${xmls[0]}"
else echo "view-model: '$arg' is not a file or directory" >&2; exit 1; fi
echo "viewing: $xml"
[ -n "${VIEW_DRYRUN:-}" ] && exit 0
exec "$APP" "$xml"
````

In AppCSXCAD: the mesh is under the grid toggle; ports show as the `port_*` probe and
excitation boxes. Inspecting the model — does each port sit on the right plane against
the right reference, does the mesh pack lines against every conductor edge (thirds rule),
is the air box / PML margin sane — is a legitimate review step before trusting a run,
and the one an RF-expert user can drive directly.

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

## Constraint register

Every consumer of the shared stackup, with the bound it imposes. Filled at Phase 0,
before any structure loop optimises. Include the EM structures **and** the
non-simulatable consumers — the packages of the already-chosen active parts, the
assembly process, and the fab. The feasible core-thickness window is the
intersection of the floor/ceiling columns; a structure's stackup-exploration
iteration is invalid if it leaves this window. "Pull" is the soft preference.

| Consumer | Floor | Ceiling | Pull | Reason (cite datasheet / rule) |
|----------|-------|---------|------|--------------------------------|
| Antenna (single element BW) | — | — | thicker | Impedance BW ∝ h/λ₀; needs <required fractional BW> |
| Dense array (λ/2 pitch) | — | <h ≲ 0.02 λ₀> | thin | Surface-wave coupling ∝ h·√εᵣ |
| Coupled-line / interdigital filters | — | <h ≲ 0.02 λ₀> | thin | Thicker → radiation, dispersion, mode rise |
| 50 Ω lines / device escape | <fab min trace> | — | thin | Too thin → narrow line, conductor loss, tolerance |
| **Active-part package (e.g. 3×3 QFN LNA/mixer)** | — | <≈0.2–0.5 mm> | **thin** | Paddle-via inductance to ground (stability/NF) + RF-pin launch must mate to pad pitch; cite the part datasheet's ground/land guidance |
| Assembly / fab | <vendor min> | <vendor max> | — | Build capability, controlled-Z tolerance |

## Decision log

The stackup is mutable; every change is a board-level event (re-verify all structures, re-run calibration). One row per decision.

| Date       | Decision                          | Rationale / evidence                          | Structures to re-verify |
|------------|-----------------------------------|-----------------------------------------------|-------------------------|
| YYYY-MM-DD | <e.g. 0.51 mm FR4 core, 1 oz Cu>  | <initial choice; or which iteration drove it> | <all / list>            |
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

The current iteration's parameters. This file mutates in place each iteration; its full state at iteration N is recoverable with `git checkout <board>/<structure>/iter-NNN`.

````yaml
iteration: 7
board: 10-ghz-rf-board               # which board this structure belongs to
structure: patch-element             # which structure's design loop this belongs to
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
  - **plots into a per-run subdir `plots/iter-NNN/`** (not the `plots/` root), so each iteration's plots survive instead of being overwritten by the next run. Default the subdir name to the iteration number; allow an override (e.g. an env var) so parameter-sweep points within one iteration land in distinct subdirs `plots/iter-NNN-<label>/`. The FDTD solve is expensive, so plots that can only be regenerated by re-running are worth keeping locally even though `plots/` is gitignored — you get the visual history without re-solving, and git stays clean.
  - **the CSX model into a per-run subdir `models/iter-NNN/<name>.xml`** via `CSX.Write2XML(...)`, mirroring the plot archive (same `iter-NNN` / sweep-label convention, also gitignored). This is the geometry + mesh + ports exactly as OpenEMS sees them; preserving it per iteration lets the user open *any past iteration* in AppCSXCAD with `view-model models/iter-NNN` to check port placement and mesh density — the two things an RF reviewer most needs to see and that the numbers alone hide. Write it just before `FDTD.Run` (so even a killed solve leaves an inspectable model). For multi-solve structures (e.g. a 2-port driven from each end) the geometry is identical across solves, so writing one CSX per iteration is enough; give distinct labels only when the geometry actually differs (e.g. a calibration vs. DUT build).
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

**`raw/summary.json` schema (both modes) — the per-run output.** Two blocks: **`criteria`** (graded — every entry has a `target` and a `pass`) and **`derived`** (ungraded diagnostics tracked across iterations, no target/pass):

````json
{
  "criteria": {
    "s11_band_max_db":     { "value": -14.2, "unit": "dB", "target": -10, "pass": true },
    "input_loss_at_f0_db": { "value":  0.18, "unit": "dB", "target": 0.5, "pass": true },
    "stability_mu_min":    { "value":  2.30, "unit": "",   "target": 1.0, "pass": true }
  },
  "derived": {
    "gain_dbi_vs_f": { "freqs_mhz": [2400, 2410, 2420, "…", 2480], "values": [2.1, 2.2, 2.3, "…", 2.0], "unit": "dBi" },
    "f_res_ghz":     { "value": 2.451, "unit": "GHz" },
    "zin_re_at_f0":  { "value": 49.2,  "unit": "ohm" }
  }
}
````

A `derived` entry is either a **scalar** `{value, unit}` or a **spectrum** `{freqs_mhz, values, unit}` for a frequency sweep. Keep `criteria` clean: do **not** park ungraded numbers there with `target: null` — a trend-plotter walking `criteria` assumes every entry has a threshold. Diagnostics (scalar or spectrum) go in `derived`.

**Band-sensitive figures span the band.** For anything the design must hold across its operating band (match, gain, efficiency, isolation), gate the criterion on the **worst case over the band** (name it `*_min_*` / `*_max_*`) and record the per-frequency sweep as a spectrum entry in `derived` so it plots vs frequency. The FDTD solve runs once; evaluating more frequencies is post-processing only (a DFT + far-field transform per point), so sample **finely** — not just the two edges and the centre, where a roll-off between samples would be missed. The cost is not runtime but record bloat: a spectrum array is far cheaper on disk than one `derived` key per frequency, yet it still grows `history.json`, so pick a step that resolves the curve without oversampling a flat response, and round the stored values. If the constant frequency grid ever dominates the record, factor it out to a single grid per structure rather than repeating it each iteration.

**Criteria evolve — promotion from `derived`.** `derived` is the watch-list: numbers you track but haven't (or can't yet) put a threshold on. When iteration evidence shows a diagnostic actually gates the design, **promote** it — move the key from `derived` into `criteria` with a target and pass test, add it to the structure's `goals.md` (the contract), and note the promotion in the iteration that introduces it. Criterion keys are **append-only**: a criterion may first appear at iteration K and be absent from earlier iterations (they predate it) — that is expected, not a gap, and a trend-plotter must tolerate it. Never rename or repurpose an existing key; introduce a new one. Relaxing or retiring a criterion (e.g. escalated as infeasible) is likewise a contract change — update `goals.md` and record it in the iteration.

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

The recorded numbers are only valid once **mesh-converged** — stable as the mesh is refined (see the rf-pcb-engineer agent's "the mesh is not the structure" principle). Refine where features are small relative to the cell (narrow gaps, feed tongues, coupled-line spacings); a feature one or two cells across is a red flag. A **convergence check is a legitimate iteration** — same geometry, finer mesh, hypothesis "is iter-N converged?"; record the figure-of-merit deltas and the verdict like any other iteration. Re-running with a changed mesh is a `model.py`/`mesh.yaml` change, so note it in the iteration record.

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

One `history.json` **per structure** — the machine-readable contract and the source for that structure's trend plots. One entry appended per iteration, carrying the `criteria` (graded) and `derived` (diagnostics) blocks from `raw/summary.json`, nested under `iterations[]`. Criterion keys are **stable and append-only** across iterations of the same structure — a criterion may first appear partway through (when promoted from `derived`); never rename or repurpose a key, and a trend-plotter tolerates keys absent in early iterations. (Different structures carry different criteria — that is why each gets its own file.)

````json
{
  "board": "<board>",
  "structure": "<structure>",
  "iterations": [
    {
      "iteration": 7,
      "parent": 6,
      "tag": "<board>/<structure>/iter-007",
      "date": "2026-05-26",
      "mode": "parametric",
      "criteria": {
        "s11_band_max_db":     { "value": -14.2, "unit": "dB", "target": -10, "pass": true },
        "input_loss_at_f0_db": { "value":  0.18, "unit": "dB", "target":  0.5, "pass": true },
        "stability_mu_min":    { "value":  2.30, "unit": "",   "target":  1.0, "pass": true }
      },
      "derived": {
        "f_res_ghz":    { "value": 2.451, "unit": "GHz" },
        "zin_re_at_f0": { "value": 49.2,  "unit": "ohm" }
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
- board: <board name, e.g. 10-ghz-rf-board>
- structure: <structure name, e.g. patch-element>
- parent: <N-1 or logical parent; null only for iter-001>
- mode: <parametric | gerber2ems>
- recover inputs: `git checkout <board>/<structure>/iter-NNN`

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
<observation, citing specific plots in plots/ — regenerate with `git checkout <board>/<structure>/iter-NNN` if needed>

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
- Design source: `<board>/kicad/` at tag `<board>/final`
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

1. **Past iterations are immutable — git enforces it.** Each iteration is a commit tagged `<board>/<structure>/iter-NNN`; never rewrite history (no `--amend`, no rebase across a published iteration). If a bug is found in iter-003's sim, create iter-004 that re-runs it correctly and explain in iter-004's markdown. Editing history breaks the story the postmortem will tell.
2. **One git repo = a PCB-design workspace, separate from software repos; one subdirectory per board.** `git init` the workspace (kept separate from any software/firmware repo so board iteration tags `<board>/<structure>/iter-NNN` don't collide with software release tags). The shared `.venv/` and `openEMS/` sit at the workspace root and are gitignored; per board, commit the scaffold (`goals.md`, `stackup.md`, `tooling.md`) before that board's first iteration. Each board's structures' `sim/`, `kicad/`, iteration records, and per-structure `history.json` are tracked together so each iteration commit captures design and simulation together. No file copying for snapshots; git is the snapshot mechanism.
3. **`raw/`, `plots/`, and `models/` are gitignored.** They are regenerable from `sim/` at any iteration tag. The numbers that must survive live in `history.json` and the iteration markdown. Do not commit field dumps, images, or CSX XML — `models/iter-NNN/<name>.xml` is a per-iteration inspection artifact (open with `view-model`), not a record; the geometry of record is `sim/` at the iteration tag.
4. **Each iteration is one commit, tagged `<board>/<structure>/iter-NNN`, with a board/structure commit message.** Format: `iter-NNN(<board>/<structure>): <hypothesis one-liner>`. The board/structure-namespaced tag is the recovery handle; the markdown stores no SHA.
5. **Zero-pad iteration numbers** (`iter-001`, not `iter-1`) in both file names and tags so sort matches chronological order. Three digits supports up to 999; if you hit that you probably need a structural rethink instead.
6. **`parent` is mandatory for iter ≥ 002.** Designs branch; the parent chain (recorded in the markdown and `history.json`) is how lineage is reconstructed, independent of git's chronological parent. Only iter-001 has `parent: null`.
7. **`history.json` is the machine-readable contract — one per structure.** Append one entry per iteration (its `criteria` + `derived` blocks) to that structure's file. Criterion keys are stable and append-only — a criterion may first appear partway through (promoted from `derived`); never rename or repurpose a key, and a plotter tolerates keys absent in early iterations. Different structures carry different criteria, which is why each has its own file.
8. **`<board>/final/` does not exist until all `[sim]` criteria pass** for that board, at which point the design source is tagged `<board>/final`. If tempted to create it early because "we're close enough", you aren't — keep iterating or escalate the criterion as infeasible.
9. **Record tool versions in `tooling.md`** and re-run the calibration reference if any version changes mid-project. Without this, "the simulation used to give X" becomes unfalsifiable.
10. **Parametric simulation models, not hardcoded geometry.** A `model.py` that rebuilds from `params.yaml` makes the next iteration a yaml edit; hardcoded coordinates make it a rewrite. Non-parametric models are allowed when geometry resists parameterization, but call them out in the iteration markdown.
11. **Parameter sweeps live inside a single iteration**, not as separate iterations. Use the `sweep` field in `sim/params.yaml` and produce one summary per swept point under `raw/sweep/<value>/` (gitignored). Record the swept results in the iteration markdown and the converged/representative point in `history.json`. One hypothesis, one iteration file, one verdict — that's what keeps the iteration log a readable narrative.
12. **No decorative formatting in templates.** Tables and plain prose. These files are read by future engineers (and future LLM context windows); make them scannable.
13. **Cast numpy scalars to Python types before `json.dump`.** Python 3.13's json rejects `numpy.bool_` and `numpy.float64`. Always wrap with `bool(...)` / `float(...)` at the JSON boundary. Symptom of forgetting: `TypeError: Object of type bool is not JSON serializable` at the very end of an otherwise-successful run.
14. **Each iteration markdown includes a "Diff vs iter-<parent>" table** when the parent exists, generated from `history.json`. Skip the section in iter-001 (no parent). Makes the iteration log readable months later.
15. **Use the μ-factor (Edwards–Sinsky) as the primary stability metric.** Rollett k is a secondary cross-check restricted to frequencies where `|S21| > −20 dB`. See "Sign conventions" above.

## First actions when invoked

1. **New board:** if the workspace repo doesn't exist, `git init` it (separate from any software repo) and set up the shared `.venv/` + `openEMS/` per README. Create the workspace-root scaffold — `.gitignore`, `README.md`, and the `view-model` launcher (`chmod +x`) — once for the workspace. Create the board subdirectory and write its `goals.md` with the user before any layout — function, signal chain, board-wide constraints, and the list of structures to design. Push back on non-measurable criteria. Refuse to start any structure's loop until every one of its criteria is tagged `[sim] / [bench] / [ext]`. Commit the board scaffold (`<board>/goals.md`, `stackup.md`, `tooling.md`; workspace `.gitignore`, `view-model`).
2. **Existing board / structure:** check the structure's `sim/params.yaml` is consistent with its `goals.md` and that the criteria-tagging is current. `git tag --list '<board>/<structure>/*'` lists that structure's iterations; its `history.json` is the numeric history.
3. **"Start a new iteration":** the working set in `<board>/structures/<structure>/sim/` is already the parent's state (it mutates in place). Edit `sim/params.yaml` — bump `iteration`, set `parent`, write the hypothesis and `changes_from_parent`, apply the geometry change — then **stop before running** so the user (or agent) can confirm the hypothesis is worth running. Do not commit yet; uncommitted changes are trivially reverted if the hypothesis is rejected. After the run, log the results, append to the structure's `history.json`, write `iterations/iter-NNN.md`, then commit and `git tag <board>/<structure>/iter-NNN`.
4. **"Present the design":** read every structure's `iterations/*.md` and `history.json` and synthesize `final/summary.md` from what's recorded. Tag the design source `final`. Do not invent narrative — use the iteration log as the source of truth.
