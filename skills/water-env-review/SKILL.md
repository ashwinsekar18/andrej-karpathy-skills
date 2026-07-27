---
name: water-env-review
description: PR review checklist for water/wastewater environmental engineering code. Covers units and physical plausibility, regulatory compliance (MCLs, permit limits), data integrity, safety-critical calculation review, and scientific reproducibility.
license: MIT
---

# Water Sector Environmental Engineering Review Guidelines

A PR review checklist for water and wastewater engineering software — analysis scripts, treatment models, SCADA integrations, compliance reporting tools, and hydraulic models.

## Why These Guidelines

Environmental engineering code in the water sector has failure modes that general software review misses:

- **Unit errors** cause incorrect dosing, misreported concentrations, and permit violations
- **Below-MDL handling** silently biases statistics when non-detects are coded as zero
- **Hardcoded regulatory thresholds** become outdated when regulations change and create quiet compliance failures
- **Unstated data quality flags** fold bad sensor readings into analysis without warning
- **Undocumented time zones** corrupt time-series analysis on data from multiple sites
- **Untested dosing calculations** can cause over- or under-treatment with direct public health impact

## The Five Areas

### 1. Units & Physical Plausibility

Water quality code lives or dies by units. Common parameters:

| Parameter | Common Units | Gotcha |
|-----------|-------------|--------|
| Concentrations | mg/L, μg/L, ng/L (ppb, ppt) | Off-by-1000x errors are silent |
| Turbidity | NTU, FNU | Not interchangeable in all contexts |
| Flow | MGD, m³/d, L/s, cfs | Mix of US customary and SI is common |
| Microbiology | cfu/100 mL, MPN/100 mL | Method affects interpretation |
| Temperature | °C, °F | Always state which |
| pH | dimensionless | Range 0–14; log scale — never average directly |

Rules:
- Every variable that carries a unit must have that unit in its name, type annotation, or an inline comment
- Conversion factors must be named constants, not inline multipliers
- Validate that inputs are physically possible before using them in calculations

### 2. Regulatory Compliance

Hardcoded thresholds go stale. Reference the rule:

```python
# BAD
if nitrate_mg_l > 10:
    raise PermitViolation(...)

# GOOD — cite the source so it can be audited and updated
# 40 CFR 141.62 — MCL for nitrate as N, effective 1992
MCL_NITRATE_AS_N_MG_L = 10.0
if nitrate_mg_l > MCL_NITRATE_AS_N_MG_L:
    raise PermitViolation(...)
```

Check:
- MCLs cite 40 CFR part and subpart (or state equivalent)
- Effluent limits cite the NPDES/state permit number and parameter table
- Analysis periods match the permit's reporting cycle (monthly, quarterly, annual)
- Technology-based limits (e.g., BAT, BCT) are distinguished from water quality-based limits

### 3. Data Integrity

Document the provenance of every data source:

- **Monitoring stations**: USGS site number, state ID, or internal tag
- **Lab data**: LIMS report ID, analytical method (EPA 300.0, SM 4500, etc.)
- **SCADA**: tag name, historian, scan rate

Non-detects require explicit handling:

```python
# BAD — biases mean low, inflates false compliance
concentration = float(raw_value.replace('<', ''))

# GOOD — flag and apply a documented substitution method
if raw_value.startswith('<'):
    mdl = parse_mdl(raw_value)
    concentration = mdl / 2  # Kaplan-Meier or ½ MDL substitution; document the choice
    flags.append('non_detect')
```

Time-series pitfalls: always store and display timestamps with explicit UTC offset; document DST handling for sites in multiple time zones.

### 4. Safety-Critical Calculations

Any code path that changes what goes into a pipe or triggers an alarm is safety-critical:

- **Dosing calculations**: chlorine residual, coagulant dose, acid/base for pH correction
- **Alarm setpoints**: turbidity spikes, pressure drops, flow deviations
- **Control logic**: pump starts, valve positions, blower speeds
- **SCADA write-back**: any code that sends commands to field devices

Requirements for these changes:
1. Independent calculation check (show the hand-calculation or cite the engineering reference)
2. Mass balance verification where applicable
3. "Fail-safe" defaults — a missing sensor reading must not silently default to zero dose or zero alarm threshold
4. Field validation sign-off before production deployment

### 5. Scientific Reproducibility

Analysis should be a pure function of its inputs:

- Pin library versions (`requirements.txt`, `environment.yml`)
- Fix random seeds for any stochastic component
- Document model version and calibration dataset
- Use appropriate statistics: water quality data is rarely normal — use log-normal, Kaplan-Meier, or non-parametric methods where appropriate
- State the detection limit and censored-data method used (substitution, MLE, K-M)
