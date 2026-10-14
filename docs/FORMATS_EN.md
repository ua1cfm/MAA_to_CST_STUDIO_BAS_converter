# Technical reference: MMANA `.maa` and Fairbeam `.design.json`

[Project README](../README.md) · [Русский справочник](FORMATS_RU.md)

This reference documents **what converter 0.3.1 actually parses and writes**, not a complete normative specification of every MMANA-GAL or Fairbeam feature. In particular, MMANA electrical source and load formats vary, and `fairbeam.design/1` compatibility should be verified against the installed Fairbeam release.

## MMANA `.maa`: structure and units

MMANA antenna files are text records. The provided samples use legacy Cyrillic headers encoded in Windows-1251, and `*`-framed section titles. The parser tries UTF-8 with BOM, Windows-1251 and Latin-1. Coordinates and wire radii are in **meters**; design frequency is in **MHz**.

A representative excerpt, with English section headings for readability, is:

```text
Example antenna
*
145.0
* Wires *
2, 1, 1, 0, 0, 0
0, 0, -0.5, 0, 0, -0.01, 0.001, -1
0, 0,  0.01, 0, 0,  0.5, 0.001, -1
*** Source ***
1, 1
w1c, 0.0, 1.0
*** Load ***
0, 1
*** Autosegm ***
400, 40, 2.0, 1
```

The example is illustrative, not proof that this exact source/port combination is electrically complete. It shows the fields recognized by the converter. Source and load records are *read for diagnostic counts only*.

### Wire header and stacking

The first wire-section record has up to six comma-separated numeric fields:

`base_wire_count, horizontal_copies, vertical_copies, horizontal_spacing_m, vertical_spacing_m, placement_mode`

For supported centered layout (mode `0`), this converter copies the base geometry along Y and Z and shifts indices of source/load records **for diagnostics only**. The parser bounds duplication to at most 64 copies and flags nonzero/unknown layout modes as unverified; it cannot establish MMANA equivalence by itself.

### Wire records

Each base conductor has eight values:

`x1, y1, z1, x2, y2, z2, radius_m, segmentation_tag`

Coordinates describe two endpoints in meters. Positive `radius_m` describes a constant-radius conductor; a negative radius can encode an MMANA combined/stepped conductor. The last field is an MMANA segmentation directive; it is **not** a CST or Fairbeam wire segment count. Converter v0.3.1 rejects unresolved stepped-radius profiles rather than silently replacing them with `abs(radius)`.

### Source and load records

A typical source record begins with a pulse reference such as `w13c` followed by phase (degrees) and excitation amplitude. End/center references (`WnB`, `WnC`, `WnE`) depend on MMANA segmentation conventions. Load records may describe capacitance, inductance, resistance, quality factor, or networks and require MMANA-specific interpretation. **None of these become live electrical objects in v0.3.1.** The `reports/*.txt` file gives the input record counts, not proof of their reconstruction.

The autosegmentation, ground conditions and combined-wire profiles in an MAA file have no automatically equivalent meaning in a geometry-only exporter.

## Fairbeam `.design.json`: exported structure

The `schema` tag used by v0.3.1 is `fairbeam.design/1`. An abbreviated representative design is:

```json
{
  "schema": "fairbeam.design/1",
  "model": { "id": "maa_example", "name": "Example antenna" },
  "params": [],
  "simulation": { "f_min": 0.13775, "f_max": 0.15225,
                  "boundaries": "MUR", "end_criteria_db": -40 },
  "materials": [{ "name": "copper", "kind": "metal" }],
  "parts": [{ "name": "antenna", "material": "copper",
    "primitives": [
      { "kind": "wire", "points": [[0,0,-500],[0,0,-10]], "radius": 1 },
      { "kind": "wire", "points": [[0,0,10],[0,0,500]], "radius": 1 }
    ]
  }],
  "ports": [],
  "resistors": [],
  "mesh": { "mode": "auto", "cells_per_wavelength": 20 },
  "far_field": { "enabled": true, "frequencies": [0.145] }
}
```

### Field reference

| Path | Meaning | Units/notes |
|---|---|---|
| `schema` | File schema identifier | Literal `fairbeam.design/1` |
| `model.id`, `model.name` | Internal ID, human-friendly name | ID is generated from the relative MAA path |
| `params` | Additional user parameters | Empty in this exporter |
| `simulation.f_min`, `f_max` | Configured frequency range | **GHz** |
| `simulation.boundaries` | Boundary configuration | `MUR` as generated; physical appropriateness must be checked |
| `simulation.end_criteria_db` | Stopping threshold | `-40` as generated |
| `materials[]` | Material definitions | `copper`, `kind: metal` |
| `parts[].primitives[]` | Geometric objects | `kind: wire` |
| `points` | Wire endpoints | Two XYZ triples, **mm** |
| `radius` | Wire radius | **mm**, not diameter |
| `ports` | Excitation ports | Intentionally empty |
| `resistors` | Lumped elements supported by the target schema | Intentionally empty |
| `mesh` | Fairbeam-side mesh settings | `auto`, 20 cells per wavelength, not MMANA segmentation |
| `far_field.frequencies` | Requested field-monitor frequency | **GHz**, central source MAA frequency |

The default frequency range is ±5% of the MAA center frequency; it is adjustable via `--sweep-percent`. Conversions: `1 m = 1000 mm`; `1 MHz = 0.001 GHz`. One `.design.json` maps one antenna file, not a complete antenna excitation definition.

## Important distinctions and validation

- **JSON syntax valid** does not imply **Fairbeam version compatible**, **mesh converged**, **port coupled**, or **simulation physically correct**.
- There is no conversion of MMANA *pulses* to Fairbeam *ports*. Likewise, an MAA load declaration does not turn into a resistor simply because the JSON schema contains `resistors`.
- Incomplete geometry is reported as an error, not silently exported. Supported stack copies result in expanded physical wires.
- `.txt` reports under `reports/` are human-readable diagnostics, never Fairbeam files; `.bas` files are CST VBA and cannot be opened as Fairbeam designs.
- For a useful electromagnetic comparison, independently configure equivalent conductors/material, excitation, load, ground, frequency and mesh in MMANA/Fairbeam, then compare impedance and radiation patterns.

## Validation workflow

1. Open `reports/<name>.txt` and ensure no geometry warning has been overlooked.
2. Compare antenna XYZ endpoints and radii after unit conversion; verify stack positioning against MMANA.
3. Confirm Fairbeam imports `*.design.json` with no schema error; record your Fairbeam application version.
4. Add power feed, loads and appropriate solver/boundary settings; check that ports contact the correct wire ends without shorts.
5. Perform mesh-convergence and frequency-range tests before trusting S11, resonances or E/H results.
6. Report issues with the original MAA, design JSON, TXT diagnostics and the exact import/solver error.
