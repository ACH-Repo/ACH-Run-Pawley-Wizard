# run-pawley-wizard

Interactive command-line wizard that generates a ready-to-run **TOPAS 7 `.inp` file** for structureless Pawley profile fitting — guided step-by-step, no manual template editing required.

---

## Quick start

```bash
pip install pymatgen
```

Navigate to the directory containing your powder data file, then run:

```bash
python run_pawley_wizard.py
```

The wizard walks you through six prompts and writes a complete `.inp` file. If TOPAS is found at the configured path it launches the fit immediately.

---

## What you'll see

```
====================================================
      Welcome to the TOPAS Input File Wizard        
====================================================
This tool will guide you step-by-step to generate a
specialized .inp template for structureless Pawley fits.

[Step 1/6] Choose your experimental powder dataset:
  [0] -> sample_A.brml
  [1] -> sample_B.raw

Select experimental file by entering its index number: 0

[Step 2/6] Reading database files from: D:\...\CIF_LOC ...

Available Crystallographic Reference Phases found:
  ZIF-4, ZIF-zni, quartz, ...

Type desired phase names (space-separated, case-sensitive): ZIF-4 ZIF-zni
  [*] Trusted starting parameters applied for: ZIF-4, ZIF-zni

[Step 3/6] Detecting instrument settings from BRML file ...
  [+] Detected: Cu anode, LP_Factor=0° (no secondary monochromator), Soller 2.5°/2.5°, Radius 280 mm
  Derived TOPAS instrument block:
    LP_Factor(!th2_monochromator, 0)
    CuKa2_analyt(0.0001)
    Radius(280)
    Full_Axial_Model(12, 20, 14, 2.5, 2.5)
    Specimen_Displacement(height,0)

Use auto-detected settings? [Y/n]:

[Step 4/6] Select the sample holder for background math operations:
  [0] -> silicon
  [1] -> plastic

[Step 5/6] Naming Strategy:
Provide custom .inp filename root (Leave blank to use data file name):

[Step 6/6] Metadata Logging:
Add a specific run note/comment to the output file header? (optional):

====================================================
     GENERATED TOPAS RUNTIME SCRIPT PREVIEW
====================================================
' ENGINE SYSTEM AUTOMATION LOG:
' Selected phases: "ZIF-4" (61) | "ZIF-zni" (110)
' Selected device: "brml_auto (...)"
...

Write template to storage and launch TOPAS execution? (y/n): y
[+] File written successfully to: 'sample_A.inp'
Launching calculation engine subprocess...
[+] TOPAS optimization run complete.
```

---

## Before you begin: two things to configure

### 1 — Point the wizard at your CIF database (`cif_dir_path`)

CIF (Crystallographic Information File) files are the standard format for crystal structure data. They contain the unit cell parameters, space group, and atomic positions that define a crystal phase. The wizard uses them to extract the lattice parameters and symmetry system for each phase you want to include in the fit.

You need a local directory populated with `.cif` files for every phase you might want to fit. A good source is the [Cambridge Structural Database](https://www.ccdc.cam.ac.uk/) or [ICSD](https://icsd.fiz-karlsruhe.de/) for inorganic structures. Each file's **stem name** (filename without `.cif`) becomes the phase name you type at step 2.

Open the script and set `cif_dir_path` near the top:

```python
SETTINGS = {
    ...
    'cif_dir_path': r'C:\path\to\your\CIF_folder',
    ...
}
```

### 2 — Set your TOPAS installation path (`tc.exe`)

TOPAS Academic (version 7) is the refinement engine. It is **not included** in this repository — it must be purchased and installed separately from [Bruker AXS](https://www.bruker.com/). `tc.exe` is the command-line driver that TOPAS Academic ships with; `run_pawley_wizard.py` calls it as a subprocess after writing the `.inp` file.

If your TOPAS installation differs from the default, edit the path near the bottom of the script:

```python
engine_executable = r"C:\TOPAS7\tc.exe"   # ← change if installed elsewhere
```

If `tc.exe` is not found the wizard still writes the `.inp` file successfully — you can then launch TOPAS manually.

---

## Configuration reference

All user-facing settings live in the `SETTINGS` dictionary at the top of the script.

| Key | Type | What it controls |
|---|---|---|
| `cif_dir_path` | `str` | Path to your CIF database folder — **must be set before first use** |
| `diffractometer` | `dict` | Named instrument profiles (LP factor, Kα₂ correction, specimen displacement). Add entries here for additional diffractometers. |
| `background` | `dict` | Pre-refined background polynomials for different sample holders (silicon zero-background plate, plastic). |
| `use_trusted_params` | `bool` | `True` (default) — use the refined cell parameters in `trusted_params` as starting values when a matching phase is selected. Set to `False` to always start from raw CIF values. |
| `trusted_params` | `dict` | Phase-name → cell-parameter overrides from previous well-converged fits. Values may carry TOPAS backtick-sigma notation (e.g. `23.45`_0.003`) to seed the refinement uncertainty estimate. Only supply the parameters the crystal system actually needs; all others remain as read from the CIF. |
| `iters` | `str` | Maximum number of refinement cycles (default `10000`). |
| `chi2_convergence_criteria` | `str` | Convergence threshold on Δ(χ²) (default `0.000001`). |
| `separator_ident` | `str` | String inserted between the output file root and the suffix of every TOPAS output file (default `_pawley_01_`). |

### Adding a diffractometer profile

Each entry in `diffractometer` is a short snippet of TOPAS syntax pasted verbatim into the `xdd` block:

```python
'diffractometer': {
    'my_instrument': (
        "\n    LP_Factor(!th2_monochromator, 0)"
        "\n    CuKa2_analyt(0.0001)"
        "\n    Specimen_Displacement(height,0)"
    ),
}
```

Key lines to configure:

| Line | When to change |
|---|---|
| `LP_Factor(!th2_monochromator, angle)` | Set `angle` to 0 for no secondary monochromator; use ~26.6° for a graphite secondary monochromator |
| `CuKa2_analyt(...)` / `CuKa2(...)` | Use `_analyt` form without a secondary monochromator; plain `CuKa2` with one. Replace prefix for other anodes (Mo, Co, Cr …) |
| `Specimen_Displacement(height, value)` | Sample-height correction; refine with `@ value` if desired |

When a `.brml` file is selected the wizard will attempt to read these settings automatically from the file (see [How it works](#how-it-works)).

---

## What the wizard produces

- A complete **TOPAS 7 `.inp` file** with:
  - Fit-quality and convergence parameters
  - `xdd` block pointing to the raw data file
  - Instrument geometry (LP factor, Kα₂ correction, axial divergence model, goniometer radius)
  - Pre-refined background polynomial for the selected sample holder
  - One `hkl_Is` block per phase with TCHZ peak-profile parameters and a crystal-system lattice macro
  - Output file declarations for observed, calculated, and difference profiles plus per-phase Bragg tick positions
  - A header comment log (run date, selected phases, device, background, any trusted-param overrides)

---

## Installation

Python ≥ 3.10 and one external package:

```bash
pip install pymatgen
```

No other dependencies beyond the Python standard library.

---

## Run from anywhere (optional)

If you want to invoke the wizard from any working directory without typing the full script path, create a one-line Windows shim file called `rp.cmd`:

```bat
@python "[FULL_PATH]\run_pawley_wizard.py" %*
```

Replace `[FULL_PATH]` with the absolute path to the folder where `run_pawley_wizard.py` lives (e.g. `C:\Users\chris\Documents\Claude\run_pawley`). Place `rp.cmd` in any folder that is on your `PATH` environment variable (e.g. `C:\Users\<you>\bin\`). After that, opening a terminal in your data directory and typing `rp` is all it takes.

---

## Project layout

```
run_pawley_wizard.py   — the wizard (single-file, all settings at the top)
README.md
.gitignore
```

---

## How it works

<details>
<summary>Click to expand</summary>

### Six-step interactive flow

| Step | What happens |
|---|---|
| 1 | Scans the working directory for `*.xy`, `*.raw`, `*.brml` data files and lets the user pick one |
| 2 | Reads every `*.cif` in `cif_dir_path` with **pymatgen**, extracts lattice parameters and crystal system, and presents the phase names. The user types one or more names; if `use_trusted_params` is on and any selected phase has an entry in `trusted_params`, those refined cell values replace the CIF values for that phase only |
| 3 | If the data file is a `.brml`, opens the ZIP archive and parses `Experiment0/MeasurementContainer.xml` to auto-detect: tube anode (`TubeMountData` → `TubeMaterial`), secondary crystal monochromator presence (any mounted component whose `BeringClassPath` contains `"monochromator"` → `LP_Factor` angle), goniometer radius, primary (`Soller/Axial/Mini`) and secondary (`Soller/Axial/DetectorAttached`) Soller angles. On success a `Full_Axial_Model` line is added and `Simple_Axial_Model` is omitted from the per-phase block to avoid double-counting. Source-focus length (12 mm), sample length (20 mm), and detector aperture (14 mm) are hardcoded Bruker D8/LYNXEYE defaults. If detection fails or the user declines, the named-profile list is shown instead |
| 4 | Selects a pre-refined background polynomial for the sample holder |
| 5 | Determines the output file root name |
| 6 | Optionally appends a free-text lab comment to the file header |

### Output file naming

All TOPAS output files share the root name with the separator string `_pawley_01_` (configurable) between the root and the suffix:

```
<root>_pawley_01_X_Yobs.txt
<root>_pawley_01_Out_X_Ycalc.txt
<root>_pawley_01_X_Difference.txt
<root>_pawley_01_2Th_Ip_<sg_num>.txt   (one per phase)
```

The separator string is how the companion [ACH-Pawley-Plotter](https://github.com/ACH-Repo/ACH-Pawley-Plotter) script discovers and groups output files.

### Trusted starting parameters

For multi-phase fits where one phase is of low symmetry or low abundance, CIF-derived lattice parameters (which may come from a structure determined at a different temperature or pressure) can put the refinement in a basin far from the true minimum. `trusted_params` lets you store cell parameters from a previous well-converged fit — including the TOPAS backtick-sigma notation — so future fits start close to the right answer without manual editing of the `.inp` file.

</details>

---

## Authorship and history

This project was originally written by **[@p3rAsperaAdAstra](https://github.com/p3rAsperaAdAstra)** as `run_pawley.py`. The original script is entirely the original author's work.

In May 2026 the script was **refactored and extended by Claude (Anthropic's AI assistant)** at the author's direction. The user-visible changes:

- BRML instrument auto-detection: reads goniometer radius, Soller angles, anode material, and secondary-monochromator presence directly from the Bruker raw-data archive, emitting `Radius`, `Full_Axial_Model`, and the correct Kα₂ macro automatically
- Trusted starting parameters: `trusted_params` / `use_trusted_params` toggle in `SETTINGS`
- `select_from_list` helper to eliminate repeated selection-loop boilerplate
- Python 3.10.0 compatibility fix (nested function return annotation)

The original `run_pawley.py` flow, template structure, background polynomials, crystal-system macros, and CIF-reading logic are unchanged from the original author's version.

This note is included for transparency about what is and isn't human-authored. The pre-rewrite version is preserved in the git history.
