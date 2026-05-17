<div align="center">

```
██╗   ██╗███████╗ ██████╗  █████╗
██║   ██║██╔════╝██╔════╝ ██╔══██╗
██║   ██║█████╗  ██║  ███╗███████║
╚██╗ ██╔╝██╔══╝  ██║   ██║██╔══██║
 ╚████╔╝ ███████╗╚██████╔╝██║  ██║
  ╚═══╝  ╚══════╝ ╚═════╝ ╚═╝  ╚═╝
```

### **Spectral Library Validation, Build & Rebuild Engine** 🌟

*Named after Vega — David Bowie's 1995 double album. The neon-star motif.*

![Python](https://img.shields.io/badge/python-3.9%2B-blue?style=flat-square)
![Platform](https://img.shields.io/badge/platform-timsTOF%20%2B%20Orbitrap-22d3ee?style=flat-square)
![Mode](https://img.shields.io/badge/mode-PROBE%20%2F%20BUILD%20%2F%20REBUILD-d946ef?style=flat-square)
![License](https://img.shields.io/badge/license-Academic-a855f7?style=flat-square)
![Part of ZIGGY](https://img.shields.io/badge/part%20of-ZIGGY-DAAA00?style=flat-square)

</div>

---

## What is VEGA?

VEGA is a **three-mode spectral library engine** that covers the full library lifecycle — from validation and health assessment through creation and continuous optimisation. It is the only tool in the ZIGGY ecosystem that treats spectral libraries as living artifacts: inspect them, build them from scratch, and iteratively improve them without ever touching the original file.

> *"The 1. Outside album was Bowie rebuilding himself — taking everything apart and putting it back together as something better. VEGA does the same for your spectral libraries."*

---

## Three Modes

```
PROBE   →  Validate an existing library — flags, health score, per-entry diagnostics
BUILD   →  Create a new library from raw .d files, FASTA, or de novo sequences
REBUILD →  Optimise an existing library — filter, fill gaps, re-score, re-predict
```

---

## PROBE — Library Validation

PROBE scores every precursor in a spectral library against a suite of quality flags.
A single health score (0–100) summarises the result. Individual flags explain exactly what is wrong.

### Flag Severity Tiers

| Tier | Colour | Flags |
|------|--------|-------|
| **CRITICAL** | red | `MISSING_CCS`, `ZERO_FRAGMENTS`, `IMPOSSIBLE_MASS` |
| **WARNING** | amber | `LOW_ION_COVERAGE`, `DUPLICATE_PRECURSOR`, `PREDICTED_LIBRARY`, `NEAR_SPECTRAL_DUPLICATE`, `CHIMERIC_OR_MISASSIGNED` |
| **INFO** | blue | `SINGLETON_PROTEIN`, `LOW_CHARGE_DIVERSITY`, `DECOY_CONTAMINATION` |

### Health Score Interpretation

| Score | Verdict | Meaning |
|-------|---------|---------|
| 90–100 | ★ Excellent | Deploy immediately |
| 70–89 | ✓ Good | Minor issues; usable |
| 50–69 | △ Marginal | Visible ID loss expected |
| 0–49 | ✗ Poor | Rebuild recommended |

### PROBE Output

| File | Description |
|------|-------------|
| `vega_report.json` | Per-entry flags, health score, summary statistics, CCS stats |
| `vega_flagged.tsv` | Entries with at least one flag (for downstream triage) |

---

## BUILD — Library Creation

BUILD constructs a spectral library from three complementary sources, then merges and deduplicates them into a single, validated `.tsv` for DIA search.

### Source Modes

| Mode | Input | Best for |
|------|-------|---------|
| **Empirical** | Bruker `.d` DDA runs + FASTA | Highest fidelity — real observed spectra |
| **Predicted** | FASTA only | No raw files needed; fast; Koina-powered |
| **Hybrid** | `.d` runs + FASTA | Fill empirical gaps with predictions — maximum coverage |
| **De novo** | `.d` runs (Casanovo) | Discover peptides outside the FASTA |

### One-Click Presets

| Preset | Mode | Charges | Conf. | CCS | Use case |
|--------|------|---------|-------|-----|---------|
| **Max IDs** | Hybrid | 2–4 | 0.65 | ✅ | Maximum peptide identifications |
| **High Quality** | Hybrid | 2–3 | 0.82 | ✅ | Fewer entries, higher confidence |
| **Quick Start** | Predicted | 2–4 | 0.70 | ✅ | No raw files — FASTA only |
| **Discovery** | De novo | 2–5 | 0.60 | ✅ | Novel peptides, non-tryptic |
| **Phospho** | Hybrid | 2–4 | 0.75 | ✅ | Phosphoproteomics |

### Koina Integration — Prediction Models

VEGA selects the best Koina model automatically based on the instrument type detected in the `.d` file:

| Instrument | MS2 model | 1/K₀ model |
|-----------|-----------|-----------|
| **timsTOF** | `Prosit_2023_intensity_timsTOF` | `IM2Deep` |
| **Orbitrap** | `Prosit_2020_intensity_HCD` | `AlphaPeptDeep_ccs_generic` |
| **QTOF** | `Prosit_2020_intensity_HCD` | `AlphaPeptDeep_ccs_generic` |

Prediction calls (MS2, RT, CCS) run **in parallel** via `ThreadPoolExecutor`. Duplicate peptide sequences are sent to Koina only once and fanned back to all entries — typically 2–4× speedup over sequential per-entry calls.

### BUILD Pipeline

```
Input sources
  │
  ├─ Empirical: .d DDA runs → TimsData SDK → PSM table (BOWIE)
  │               → real MS2 spectra + observed RT + measured 1/K₀
  │
  ├─ Predicted: FASTA digest → Koina API
  │               → predicted MS2 + predicted RT + predicted 1/K₀
  │
  ├─ De novo:  .d runs → Casanovo → novel sequences
  │               → Koina MS2/RT/CCS predictions
  │
  ├─ Merge + deduplicate (empirical wins on conflict)
  │
  ├─ CCS correction (run-specific calibration polynomial)
  │
  ├─ PROBE validation → flag and health-score the finished library
  │
  └─ Write: library.tsv  +  vega_provenance.json
```

---

## REBUILD — Library Optimisation

REBUILD takes any existing spectral library (regardless of origin) and applies a pipeline of targeted improvements — without ever modifying the original file.

### What REBUILD Does

| Step | Action |
|------|--------|
| **Filter bad entries** | Remove CRITICAL-flagged precursors |
| **Remove singletons** | Drop proteins with only one precursor |
| **Fill missing 1/K₀** | Predict CCS for entries without ion mobility via IM2Deep |
| **Apply CCS correction** | Re-calibrate 1/K₀ values using a run-specific polynomial |
| **Re-predict MS2** | Replace fragment intensities with fresh Koina predictions |
| **PROBE validation** | Score the output — report improvement |

### One-Click Presets

| Preset | Filters bad¹ | Strict quality² | Fill 1/K₀ | Apply CCS | Re-predict | Use case |
|--------|-------------|-----------------|----------|----------|-----------|---------|
| **⚡ Max IDs** | ✅ | ❌ | ✅ | ✅ | ❌ | Maximise recoverable entries |
| **◈ High Quality** | ✅ | ✅ | ✅ | ✅ | ❌ | Remove noise, maximise confidence |
| **K₀ CCS Native** | ❌ | ❌ | ✅ | ✅ | ❌ | Add 1/K₀ to a 2D library |
| **✦ Discovery** | ❌ | ❌ | ✅ | ✅ | ❌ | Preserve every entry, add CCS |
| **↺ Full Refresh** | ✅ | ❌ | ✅ | ✅ | ✅ | Complete re-prediction from Koina |

¹ **Filters bad** — removes entries with 0–2 peaks (empty / too-few-peaks) and chimeric near-duplicates detected by spectral fingerprint matching.  
² **Strict quality** — additionally removes marginal 3–4 peak entries. For `.blib` / `.msp` libraries with observation copy-count data, also removes singletons (peptides seen only once across MS runs).

### Safe Output — Never Overwrites

REBUILD **always** writes to a new file. If the output name matches the input, a timestamp is appended automatically:

```
my_library.tsv  →  (input, unchanged)
my_library_20250516_143022.tsv  →  (rebuilt output)
```

---

## Quick Start

```bash
git clone https://github.com/MKrawitzky/VEGA.git
cd VEGA
pip install -r requirements.txt
```

### PROBE — validate an existing library

```python
from vega import LibraryValidator

report = LibraryValidator("E:/libs/human_dia.tsv").validate()
print(f"Health: {report.health_score}/100 — {report.verdict}")
for flag, count in report.flag_counts.items():
    print(f"  {flag}: {count}")
```

### BUILD — create a new library

```python
from vega import LibraryBuilder

lib = LibraryBuilder(
    raw_paths  = ["E:/data/run1.d", "E:/data/run2.d"],
    fasta_path = "E:/assets/human.fasta",
    preset     = "max_ids",        # max_ids | high_quality | quick | discovery | phospho
    output_dir = "E:/libs/",
    output_name = "human_new.tsv",
)
result = lib.build()
print(f"{result.n_precursors} precursors → {result.tsv_path}")
```

### REBUILD — optimise an existing library

```python
from vega import LibraryRebuilder

rebuilt = LibraryRebuilder(
    input_path  = "E:/libs/human_dia.tsv",
    preset      = "high_quality",  # max_ids | high_quality | ccs_native | discovery | full_refresh
    output_dir  = "E:/libs/",
    output_name = "human_dia.tsv", # safe — timestamp appended if name conflicts
).rebuild()

print(f"Health: {rebuilt.health_before} → {rebuilt.health_after}  (+{rebuilt.improvement_pct}%)")
print(f"Entries: {rebuilt.n_precursors_in} → {rebuilt.n_precursors_out}")
print(f"CCS filled: {rebuilt.ccs_filled}  |  MS2 re-predicted: {rebuilt.ms2_repredicted}")
```

### Via ZIGGY dashboard

VEGA is fully integrated into the [ZIGGY](https://github.com/MKrawitzky/Ziggy) dashboard — use the **VEGA** tab to PROBE, BUILD, or REBUILD any library without writing a line of code.

---

## Output Files

| File | Description |
|------|-------------|
| `library.tsv` | Final spectral library in DIA-NN / Spectronaut format |
| `vega_report.json` | PROBE flags, health score, CCS stats, per-entry diagnostics |
| `vega_flagged.tsv` | Flagged entries only (for triage) |
| `vega_provenance.json` | All build/rebuild parameters, Koina models, timing, versions |

---

## Architecture

```
vega/
├── __init__.py          ← LibraryValidator, LibraryBuilder, LibraryRebuilder public API
├── validator.py         ← PROBE — flag engine + health score
├── flags.py             ← Flag definitions, severity tiers, descriptions
├── builder.py           ← BUILD — empirical / predicted / hybrid / de novo
├── rebuilder.py         ← REBUILD — filter → fill → correct → re-predict → validate
├── koina.py             ← Koina REST client: parallel calls, deduplication, model defaults
├── digest.py            ← Tryptic / non-specific digest + mod enumeration
├── ccs_correction.py    ← Run-specific CCS calibration polynomial
├── presets.py           ← BUILD_PRESETS + REBUILD_PRESETS
└── writer.py            ← TSV + provenance JSON output
```

---

## Requirements

- Python 3.9+
- `numpy`, `scipy`, `polars`
- `requests` (Koina API)
- Bruker TimsData SDK (for empirical / de novo BUILD modes)
- Optional: `casanovo` (de novo BUILD mode)
- Optional: Koina endpoint (predicted / hybrid / REBUILD modes)

---

## Part of the ZIGGY Ecosystem

VEGA is one of six in-house engines inside [ZIGGY](https://github.com/MKrawitzky/Ziggy):

| Engine | Mode | Approach |
|--------|------|----------|
| **VEGA** | Library | PROBE · BUILD · REBUILD · Koina-powered predictions |
| BOWIE | DDA + DIA | 4D database search · presets · open search |
| GauDIA | DIA | Fragment index + cosine + BH-FDR |
| Copperfield | DIA | GauDIA + CCS gate + Kálmán RT + Percolator |
| PHANTOM | DIA | Particle filter + EKF + GRU + EB-FDR |
| Goya | DDA | SGD logistic · 8 features · BH-FDR |

---

## License

**ZIGGY / STAN Academic License** — Copyright © 2024–2026 Michael Krawitzky

**Free for:** academic research · non-profit · education · government-funded research · core facility internal QC

**Commercial use requires a license:** for-profit companies · CROs & pharma · fee-for-service  
Contact: bsphinney@ucdavis.edu

---

## Author

**Michael Krawitzky** — The Peptide Wizard · Bruker Daltonics  
[github.com/MKrawitzky](https://github.com/MKrawitzky)

---

<div align="center">

*"I'm deranged… I'm deranged… down, down, down…"*

🌟 **VEGA** — where spectral libraries meet Outside 🌟

</div>
