# MAA → CST Studio Suite 2026 / Fairbeam

**Version 0.3.1 — Dual Geometry Export · © UA1CFM, 2026**

[Русская версия](README_RU.md) · [Technical format reference](docs/FORMATS_EN.md) · [Описание форматов на русском](docs/FORMATS_RU.md)

A Windows/Python converter for **MMANA-GAL antenna models (`.maa`)**. It exports **CST Studio Suite 2026 VBA geometry macros (`.bas`)** and/or **Fairbeam geometry designs (`.design.json`)**. Both outputs are generated from the same parsed wire list. Batch conversion preserves the input directory hierarchy and produces plain-text diagnostics.

> **Scope: geometry only.** Version 0.3.1 does **not** transfer sources, ports, lumped R/L/C loads, ground models, conductor losses or the MMANA current solution. An exported design must not be interpreted as electrically equivalent to the original model. Fairbeam JSON structural checks do not establish successful simulation in an installed Fairbeam release.

## Main features

- Choose **CST BAS**, **Fairbeam JSON**, or **both** in the Tkinter GUI or CLI.
- Convert individual files or recursively process folder trees, preserving subfolders.
- Read the model frequency (MHz), coordinates (meters), wire radius (meters), and stack header from the source MAA.
- Expand supported MMANA stacked arrays into separate physical wires; record unverified placement conventions in diagnostics.
- CST: build complete uncut conductors using `Polygon3D` followed by `Wire` (`CurveWire`); set geometry units to mm, frequency units to GHz, and configure a default ±5% frequency range and frequency-domain sweep.
- Fairbeam: output `fairbeam.design/1` with `parts[].primitives` wire geometry, metal material, frequency range, optional solver metadata, **empty** `ports` and `resistors` arrays.
- Generate UTF-8 plain-text diagnostics under `reports/` and a `BATCH_REPORT.csv` summary. Progress percentage and a visible Start Conversion button are provided.
- **No silent geometric fabrication:** combined/stepped wires with an unresolved negative-radius profile, zero-length wires and incomplete geometry cause a rejected export rather than a substitute geometry.

## Installation and run (Windows)

Requirements: Python **3.9+** with Tkinter (normally included in the Windows installer). Extract the release ZIP and run `RUN_GUI_WINDOWS.bat`. Choose a source `.maa` file or folder, destination, output formats and sweep percentage; press **Start Conversion**. The converter does not install or launch CST/Fairbeam.

### Command line

```bat
py -3 maa_to_cst2026.py --batch C:\AntennaModels --outdir C:\Converted --format both --sweep-percent 5
py -3 maa_to_cst2026.py --input C:\AntennaModels\dipole.maa --outdir C:\Converted --format fairbeam
py -3 maa_to_cst2026.py --input C:\AntennaModels\dipole.maa --output C:\Converted\dipole.bas --format cst
```

`--format` accepts `cst`, `fairbeam` and `both`. The `--sweep-percent` parameter is the half-width of the analysis range relative to the MAA working frequency. The default of 5 means 0.95 × f₀ to 1.05 × f₀. The original source file is not modified.

To build a stand-alone Windows executable, run `BUILD_EXE_WINDOWS.bat` (PyInstaller is required). An EXE build is platform-specific; it does not include CST or Fairbeam.

## Output layout

```text
Converted/
  ArrayA.bas                    # CST macro, when selected
  ArrayA.design.json            # Fairbeam design, when selected
  reports/ArrayA.txt            # diagnostics only — not a Fairbeam design
  BATCH_REPORT.csv              # batch status per antenna
  FAILED_AND_WARNINGS.txt       # exceptions and warnings
  NestedFolder/ArrayB.bas
  NestedFolder/ArrayB.design.json
  reports/NestedFolder/ArrayB.txt
```

### Opening the results

**CST Studio Suite 2026:** run the `.bas` macro in a suitable Microwave Studio project. The macro creates wires, but **does not delete existing geometry**, configure the mesh, add excitation or simulate on its own. Review duplicate geometry before re-running.

**Fairbeam:** open the `.design.json` as a design file in a Fairbeam build supporting the `fairbeam.design/1` schema. Before simulation, add and validate appropriate sources, contact geometry and electrical loads in Fairbeam. Version 0.3.1 Fairbeam opening and solver execution have not been independently verified.

## Source MAA → output correspondence

| MMANA source | CST `.bas` | Fairbeam `.design.json` |
|---|---|---|
| Antenna title | Source comments | `model.name` |
| Working frequency, MHz | `Solver.FrequencyRange` and sweep settings, GHz | `simulation.f_min/f_max`, GHz; `far_field.frequencies` |
| Endpoints, m | `Polygon3D.Point`, mm | `primitives[].points`, mm |
| Radius, m | `Wire.Radius`, mm | `primitives[].radius`, mm |
| Stacking metadata | Physical copies (supported layouts) | Same physical copies |
| MAA sources / phase | Not exported | `ports: []` |
| MAA loads / RLC | Not exported | `resistors: []` (no components) |
| Ground / MMANA pulse segmentation | Not reproduced | Not reproduced |

Please read the [detailed format reference](docs/FORMATS_EN.md) before importing engineering-critical designs.

## Batch status and validation

The project’s archived **1,713-model corpus** run for the strict dual export reported **1,105 OK**, **2 WARNINGS** and **606 ERROR**. These count converter export outcomes, **not** successful CST/Fairbeam electromagnetic solves. Warnings or errors must be inspected on each affected model; do not treat the overall OK count as a numerical accuracy benchmark.

Examples bundled in the release include `4xQQ` and `hornYagi-2`. Both exported formats were checked for basic structural consistency and matching wire counts in the conversion workflow. The result does not validate impedance, SWR, currents, gain, polarization or radiation patterns.

## Known limitations

1. **Combined/stepped conductors:** some MMANA wires use a signed negative radius to refer to a piecewise diameter profile. Until those profiles are reconstructed, strict export rejects the antenna.
2. **Array layout:** supported stacks are expanded using a centered copy layout; unusual MMANA placement modes may not match MMANA's own placement. The report calls this out where detectable.
3. **Electrical topology:** geometry alone cannot reproduce feeding gaps, loads, current phases, earth images or radiation behavior. No source or load is fabricated.
4. **Conductor material:** the Fairbeam geometry specifies the material entry `copper` with `kind: metal`; actual physical loss modeling depends on the destination application's interpretation and solver settings.
5. **CST solver setup:** the converter specifies a frequency-domain range/sweep but leaves the CST mesh manual. No solver run or mesh convergence is performed by the converter.
6. **Fairbeam compatibility:** JSON syntax and `fairbeam.design/1`-shaped data are checked, but full compatibility is specific to the Fairbeam version. Import and simulation must be validated in the application.
7. **No exact MMANA segmentation:** MMANA segment counts and auto-segmentation rules are not reproduced by this geometry-only export.

## Troubleshooting

- **No output file:** consult `reports/<model>.txt` and `FAILED_AND_WARNINGS.txt`; unsupported geometry is intentionally rejected.
- **Fairbeam cannot open a JSON:** ensure that the file ends in `.design.json`, starts with `"schema": "fairbeam.design/1"`, and is not a diagnostic TXT or a legacy `.audit.json`. Include the exact Fairbeam error in a bug report.
- **Geometry differs from MMANA:** compare endpoints/radii in mm and inspect the stack mode or signed-radius warning before trying a workaround.
- **CST creates duplicates:** open a new project or manually clear previous geometry; this release deliberately has no destructive reset command.
- **No S11 in Fairbeam:** there are no excitation ports; add valid ports in the destination software.

## Project files

`maa_to_cst2026.py` — parser, converters, CLI and GUI; `RUN_GUI_WINDOWS.bat` — launcher; `BUILD_EXE_WINDOWS.bat` — PyInstaller helper; `sample_maa/` — source examples; `examples/` — generated BAS/JSON and reports.

## License, attribution and reporting

**Author:** UA1CFM · **Copyright © 2026 UA1CFM**. This README does not grant a software license; publish an explicit `LICENSE` for your own code before making the repository public and respect the licenses of MMANA-GAL, CST, Fairbeam, and any redistributed example antenna designs. Trademarks belong to their owners; this is an independent converter, not an official integration.

For reproducible bug reports attach the original `.maa`, the generated `.bas`/`.design.json`, the matching `reports/*.txt`, converter version, target application version, and the exact error. Avoid reporting generated JSON as a validated simulated antenna merely because it parses.
