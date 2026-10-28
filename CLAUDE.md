# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project: Graphormer-AR

Research project probing **state-dependent predictability of West Coast Atmospheric River (AR) activity** using a Graph Transformer.

### Goal

Understand how large-scale climate states (MJO phase, ENSO, PDO, etc.) modulate subseasonal AR predictability on the US West Coast at leads of 1–4 weeks. This is an exploratory research effort, not targeting operational deployment.

### Model architecture

A Graph Transformer with:
- **Embedding MLP** — encodes node features into a latent space
- **Biased multi-head attention blocks** — initial attention bias set by locality/correlation priors; updated by context
- **Feedforward layer** (×3)
- **Attention readout** — aggregates node representations for prediction

### Graph structure

- ~60 **mesh nodes** covering the North Pacific, each with regionally and 7-day averaged features: Z500, IVT_x, IVT_y, and their tendencies
- **Coastal nodes** representing West Coast AR landfall points
- **Climate-mode tokens**: RMM1, RMM2 (MJO), QBO, ENSO, AO, PDO, IOD, PNA, NPO, DOY, Lag

The graph is **fully connected**; attention weights govern effective node-interaction strength given the current climate context.

### Target / output

AR frequency anomaly **terciles** (above / near / below normal) on coastal nodes at forecast leads of **weeks 1, 2, 3, and 4**, with calibrated prediction confidence. Also outputs the Z500 regime in the North Pacific.

### Analysis workflow

1. Save inference on held-out winter evaluation data: forecasts, confidence scores, attention matrices, gradients
2. Partition by global state (e.g. MJO × ENSO, MJO × PDO) or confidence level
3. Compute **layer-wise attention composites** to identify which source nodes (tropics, ENSO token, RMM tokens, etc.) drive predictions at each lag
4. **Gradient-based causal attribution** — sensitivity maps showing relative impact of information sources
5. **Ablation studies** — remove regions or climate modes and retrain to quantify skill loss

## Environment

Python environment: `/global/homes/p/pkooloth/.conda/envs/MLenv/bin/python3`

This project runs on NERSC (Perlmutter). Scratch data lives under `/pscratch/sd/p/pkooloth/`. 

## Data

Data source: `/global/cfs/cdirs/m1867/ccchang3/Parvathi_LENS2` (CESM2-LENS2 historical, 1920–2014, ~85 TB)

### Experiments

- `BHISTcmip6/` (~66 TB): historical run with CMIP6 forcing, 50 members
- `BHISTsmbb/` (~19 TB): historical run with smoothed biomass-burning forcing, 50 members

Both have the same member layout: 10 single members (`LE2-1001`–`LE2-1181` for cmip6, `LE2-1011`–`LE2-1191` for smbb) plus four groups of 10 (`LE2-1231`, `1251`, `1281`, `1301`). Member folders are named `b.e21.<exp>.f09_g17.LE2-<id>.<nnn>`.

### Per-member files

Each member folder holds one file per variable per decade (e.g. `...cam.h1.Z500.19200101-19291231.nc`):
- `FLUT` (OLR proxy), `PS`, `TS`, `Z500`
- `U`, `V`, `Q` on model levels; `U_plev`, `V_plev`, `Q_plev` on pressure levels
- `U200`, `V200`, `U850`
- `cam.h1` (70 files, likely daily) and `cam.h6` (60 files, likely sub-daily); unverified

No IVT variable is stored; compute IVT from `U`, `V`, `Q`.

Per-member subfolders: `RMM_indices/`, and `pressure_interp/` in some members.

### Helper folders

- `Codes_for_RMM/`: NCL/Python for MJO RMM EOFs and anomalies; `MJO.sh` driver
- `YYYYMMDD_for_RMM_indices/`: 96 calendar lookup files (`1920MMDD.txt`, …)
- `RMM_pattern_correlations/`: FLUT pattern-correlation Excel files (May–Oct, Jan–Apr/Nov–Dec; 30°S–30°N; 100 members vs ensemble mean)
- Top-level MJO FLUT composite `.nc` files (summer and winter, amp ≥ 1, 100-member mean)
- `Do_BHISTcmip6.sh`, `Do_BHISTsmbb.sh`: run RMM calculations per group