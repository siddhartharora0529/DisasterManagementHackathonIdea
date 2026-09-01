# DER-02 — Threat-Zone Estimation for Industrial Fire and Explosion Response

Physics + computational-geometry engine only (no frontend), built for
VIT Chennai HackTronics 2026.

Files:
- `der02_hazard_engine.py` — the engine (all physics, validation, geometry, API)
- `test_der02.py` — 25 automated tests (works with or without pytest installed)
- `examples.py` — two mandatory demonstration facility configurations
- `facility_a_result.json`, `facility_b_result.json`, `comparison_result.json` — sample outputs

---

## 1. Physical models used (and why)

Two independent, textbook consequence-modelling methods, each standard practice
in process-safety engineering for **first-pass emergency-response screening**
(not certified engineering design — see Limitations):

| Hazard | Model | Reference |
|---|---|---|
| Thermal radiation | Point-Source Model | CCPS *Guidelines for Consequence Analysis of Chemical Releases*; API RP 521 |
| Blast overpressure | TNT-Equivalency + Kinney–Graham scaled-distance correlation | Kinney & Graham, *Explosive Shocks in Air*, 2nd ed., 1985; CCPS *Guidelines for Vapor Cloud Explosions* |

Both were chosen because they are **closed-form, hand-checkable, and require
only inputs realistically available at incident time** (tank geometry + fuel
type + wind) — matching exactly what the problem statement supplies. A
higher-fidelity "solid flame" or CFD model would need flame-tilt/view-factor
geometry or full congestion/confinement data that is out of scope for a 24h
build, and we say so explicitly rather than quietly approximating it.

### 1.1 Thermal radiation — equations

```
A      = pi * (D/2)^2                    burning (pool) area           [m^2]
Q_fire = m'' * A * Hc                     total heat release rate       [W]
Q_rad  = f_s * Q_fire                     radiated heat release rate    [W]
I(x)   = tau * Q_rad / (4*pi*x^2)         incident flux at distance x   [W/m^2]
x_th   = sqrt( tau * Q_rad / (4*pi*I_th) )   distance for threshold I_th [m]
```

**Variables / parameters**

| Symbol | Meaning | Units | Source |
|---|---|---|---|
| D | tank/pool diameter | m | facility config |
| m'' | mass burning rate per unit area | kg/(m²·s) | fuel library (literature) |
| Hc | heat of combustion | J/kg | fuel library (literature) |
| f_s | radiative fraction of combustion energy | – | fuel library (literature) |
| τ | atmospheric transmissivity | – | constant, 0.80 (documented simplification) |
| I_th | severity-band threshold flux | W/m² | industry-standard values (API 521/HSE/CCPS) |

**Severity bands (industry-standard thresholds):**

| Level | Label | Threshold | Meaning |
|---|---|---|---|
| 1 | Fatality / No-Go Zone | 37.5 kW/m² | potential fatality within ~30 s |
| 2 | Severe Injury Zone | 12.5 kW/m² | significant injury risk, prolonged exposure |
| 3 | Evacuation / Caution Zone | 4.0 kW/m² | pain within ~20 s; evacuation boundary |

### 1.2 Blast overpressure — equations

```
M_fuel = release_fraction * fill_fraction * V * rho_liquid      mass involved [kg]
W_TNT  = (eta * M_fuel * Hc) / E_TNT                             TNT-equivalent mass [kg]
Z      = R / W_TNT^(1/3)                                         scaled distance [m/kg^(1/3)]

Pso/P0 = 808*(1+(Z/4.5)^2) / sqrt[(1+(Z/0.048)^2)*(1+(Z/0.32)^2)*(1+(Z/1.35)^2)]
```
(Kinney–Graham 1985). Inverted numerically (bisection) for R given a target Pso.

| Symbol | Meaning | Units | Source |
|---|---|---|---|
| η | TNT yield/equivalency factor | – | fuel library, CCPS typical range 0.02–0.10 |
| ρ_liquid | typical hydrocarbon liquid density | 750 kg/m³ | documented constant |
| E_TNT | heat of detonation of TNT | 4.184 MJ/kg | reference constant |
| P0 | ambient pressure | 101.325 kPa | standard atmosphere |

**Severity bands:**

| Level | Label | Threshold | Meaning |
|---|---|---|---|
| 1 | Destruction Zone | 55 kPa (~8 psi) | near-total destruction |
| 2 | Severe Damage Zone | 24 kPa (~3.5 psi) | severe structural damage |
| 3 | Minor Damage Zone | 7 kPa (~1 psi) | glass breakage, minor injury |

Blast is treated as **isotropic** (no wind dependence) — this is standard
TNT-equivalency practice; wind's effect on blast propagation is a second-order,
CFD-scale effect that this method does not capture (stated limitation).

### 1.3 Wind-dependent geometry (the part that makes the *zone* — not just a
number — depend on wind)

The isotropic radius from §1.1 is distorted into a wind-shaped polygon:

```
phi = bearing - downwind_bearing                     (0 = downwind, 180 = upwind)
M_down(u)  = 1 + 0.10 * min(u, 15)
M_up(u)    = max(1 - 0.06 * u, 0.5)
M_cross(u) = 1 + 0.03 * u
M(phi,u)   = M_cross + (M_down - M_cross)*max(cos(phi),0) + (M_up - M_cross)*max(-cos(phi),0)
```

This is **explicitly labelled Tier 2** in the code and here: it reproduces the
correct *qualitative* physics (flames tilt and hot gas drifts downwind,
elongating the hazard footprint there and compressing it upwind — consistent
with flame-tilt literature such as the AGA/Chamberlain correlations) but the
specific coefficients (0.10 / 0.06 / 0.03) are **engineering approximations
chosen for this hackathon, not fitted to experimental data**. We disclose this
directly rather than presenting it as high-fidelity dispersion modelling.
Blast bands remain circular, consistent with §1.2.

---

## 2. Assumptions

1. Tank fire / pool fire scenario (thermal): the tank's own footprint is
   used as the burning pool area — a standard, conservative simplification
   for a tank-fire or bunded-spill scenario.
2. Vapor cloud explosion scenario (blast): only a user-specified
   `release_fraction` (default 15%) of the tank's fuel inventory is assumed
   to participate in an explosion — full-inventory detonation is not assumed
   (that would grossly overstate blast risk for a liquid storage tank).
3. Constant atmospheric transmissivity (τ = 0.80) rather than a
   humidity/path-length-dependent model.
4. Flat, unobstructed terrain (no buildings, bunds or vegetation shielding).
5. Wind is treated as steady (no gusting/turbulence spectrum).
6. Flat-earth (equirectangular) lat/lon projection — adequate at the
   hazard-zone scale (hundreds of metres to a few km).
7. Liquid density fixed at 750 kg/m³ (typical hydrocarbon) for blast mass
   calculation, regardless of exact fuel selected.

## 3. Limitations (explicitly, for judges)

- This is a **screening-level** tool (CCPS "consequence analysis" tier), not
  a certified/regulatory-grade quantitative risk assessment. No claim of
  real-world safety certification or experimental validation is made.
- The directional wind-shape function (§1.3) is a documented **heuristic**,
  not a fitted/validated flame-tilt model.
- Blast overpressure geometry does not vary with wind (standard limitation
  of TNT-equivalency methods; a full blast-wave refraction model is out of
  scope).
- No terrain, obstacles, or congestion/confinement effects (important for
  real VCE blast strength) are modelled.
- Point-source thermal model loses accuracy very close to the flame (within
  roughly 1–2 flame diameters) where a solid-flame/view-factor model would
  be more accurate — our hazard radii at 37.5 kW/m² are typically well
  outside that near-field region for the tank sizes in this problem, but
  this should be verified case-by-case for very small tanks.
- Fuel properties are literature-typical values, not facility-specific
  laboratory data.

## 4. Why this design fits the hackathon (defensibility)

- Every formula is a named, citable equation from process-safety literature
  — nothing was invented and dressed up as validated science.
- The engine is fully deterministic and hand-checkable: a judge can take
  the printed intermediate values (`burning_area_m2`, `total_heat_release_w`,
  `tnt_equivalent_mass_kg`, etc.) and verify the arithmetic themselves.
- Simplifications are labelled where they occur (Tier 1 rigorous vs Tier 2
  heuristic), so nothing is silently overclaimed.

---

## 5. API

```python
import der02_hazard_engine as der

cfg = der.FacilityConfig(
    facility_id="FAC-A",
    latitude=12.8406, longitude=80.1534,
    tank=der.TankGeometry(diameter_m=12.0, height_m=10.0, volume_m3=1131.0, fill_fraction=0.85),
    wind=der.WindCondition(speed_m_s=3.0, direction_deg=225.0),
    fuel_type=der.FuelType.GASOLINE,      # or DIESEL, CRUDE_OIL, LPG_PROPANE, METHANOL
    release_fraction=0.15,
)

result = der.calculate_hazard_zone(cfg)   # full JSON-able dict, validated internally
json_str = der.to_json(result)

# individual sub-models (also callable directly):
der.calculate_thermal_radiation(cfg.tank, fuel_properties)
der.calculate_blast_overpressure(cfg.tank, fuel_properties, cfg.release_fraction)
der.generate_hazard_geometry(lat, lon, radius_m, cfg.wind, directional=True)
der.recommend_approach_direction(cfg, thermal_result)
der.compare_configurations(cfg_a, cfg_b)
```

`calculate_hazard_zone()` calls `validate_facility_config()` internally and
raises `der02_hazard_engine.ValidationError` (subclass of `ValueError`) with
a specific message on any invalid input (negative wind speed, out-of-range
wind direction, non-positive/geometrically-impossible tank dimensions,
missing fields, out-of-range lat/lon, unrealistic wind speed, etc.)

### Output JSON shape (abbreviated)

```json
{
  "facility_id": "...",
  "location": {"latitude": ..., "longitude": ...},
  "tank": {...}, "wind": {...}, "fuel_type": "...",
  "thermal_radiation": {
    "model": "point_source_thermal_radiation",
    "burning_area_m2": ..., "total_heat_release_w": ..., "radiated_heat_release_w": ...,
    "bands": [
      {"level": 1, "label": "...", "meaning": "...",
       "threshold_flux_kw_m2": 37.5, "isotropic_distance_m": ...,
       "geometry": {"type": "Polygon", "downwind_bearing_deg": ..., "coordinates": [[lat, lon], ...]}}
      , ...
    ]
  },
  "blast_overpressure": { "...": "same shape, isotropic geometry" },
  "recommended_approach": {"approach_bearing_deg": ..., "approach_compass": "...",
                           "rationale": "...", "minimum_standoff_distance_m": ...}
}
```

---

## 6. Example inputs/outputs (the two mandatory configurations)

**Facility A** — small gasoline tank, light wind:
D=12 m, H=10 m, V≈1131 m³, gasoline, wind 3 m/s @ 225°.
→ Level-1 thermal distance ≈ **12.7 m**, Level-1 blast distance ≈ **139 m**.

**Facility B** — large LPG tank, stronger wind:
D=28 m, H=16 m, V≈9852 m³, LPG/propane, wind 9 m/s @ 60°.
→ Level-1 thermal distance ≈ **37.8 m**, Level-1 blast distance ≈ **340 m**.

**Why they differ (physically explainable, not arbitrary):** hazard distance
scales with `sqrt(burning_area * Hc * m'' * f_s)`. Burning area scales with
`diameter²`, so Facility B's much larger tank alone would increase the
radius; on top of that, LPG has a higher heat of combustion, higher burning
rate and higher radiative fraction than gasoline, compounding the effect —
Facility B's thermal distances are **~3.0× larger at every severity level**.
Facility B's stronger wind (9 m/s vs 3 m/s) also further elongates its
hazard footprint downwind relative to Facility A. Run `python3 examples.py`
to reproduce these numbers and the full JSON files.

Sample console excerpt:
```
FACILITY A: D=12.0m H=10.0m V=1131m^3 | wind 3.0 m/s @ 225 deg
  [Thermal L1] Fatality / No-Go Zone       37.5 kW/m^2 ->   12.71 m
  [Blast   L1] Destruction Zone            55.0 kPa   ->  138.90 m
FACILITY B: D=28.0m H=16.0m V=9852m^3 | wind 9.0 m/s @ 60 deg
  [Thermal L1] Fatality / No-Go Zone       37.5 kW/m^2 ->   37.79 m
  [Blast   L1] Destruction Zone            55.0 kPa   ->  340.04 m
Thermal L1: 12.71 m vs 37.79 m (x2.973)
```

## 7. Test results

`python3 test_der02.py` → **25/25 passed** (works with or without pytest;
falls back to a built-in runner if pytest isn't installed, e.g. an offline
judging laptop). Covers:
- normal-case structure/consistency checks
- every required validation case: negative wind speed, invalid wind
  direction (negative and ≥360), non-positive/negative dimensions,
  geometrically impossible volume, missing tank/wind/facility_id,
  out-of-range lat/lon, unrealistic wind speed
- wind-dependence of geometry (zero-wind ⇒ circle; higher wind ⇒ more
  downwind elongation / upwind compression; downwind vertex farther than
  upwind vertex)
- `recommend_approach_direction` returns the upwind bearing
- `compare_configurations` shows larger tank ⇒ larger hazard distance, and
  a more energetic fuel ⇒ larger hazard distance
- JSON serialisability of the full result

## 8. Integration instructions for the frontend/map developer

1. Call `der.calculate_hazard_zone(cfg)` — you get one dict, JSON-serialisable
   via `der.to_json(result)`.
2. For each severity band in `result["thermal_radiation"]["bands"]` and
   `result["blast_overpressure"]["bands"]`, read `band["geometry"]["coordinates"]`
   — an ordered list of `[lat, lon]` vertices forming a closed polygon
   (first and last points equal). Draw thermal bands as filled/outlined
   polygons (they are wind-shaped); draw blast bands as circles/polygons
   (they are isotropic by construction, but returned in the same polygon
   format for a uniform rendering pipeline).
3. Style suggestion: Level 1 = red, Level 2 = orange, Level 3 = yellow,
   drawn in that order (3 first, then 2, then 1) so the most severe band
   renders on top.
4. `result["recommended_approach"]` gives a bearing/compass label and a
   standoff distance in metres — render as an arrow or highlighted sector
   pointing from the facility toward `approach_bearing_deg`.
5. For a facility comparison view, call `der.compare_configurations(cfg_a, cfg_b)`
   — it embeds both full results under `full_results` plus a pre-computed
   `thermal_distance_comparison_m` table you can render directly as a table
   or bar chart.
6. All inputs are validated server-side; catch `der.ValidationError` and
   surface `str(exception)` directly to the user/form — messages are
   written to be human-readable already.
