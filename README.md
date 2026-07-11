# Reachability — Replication Package (Brief Communication)

Replication code and data for **"No allocation of the remaining carbon budget is physically reachable for all regions: a feasibility screen for climate-policy proposals."**

The result is closed-form. Each region has a feasibility floor `aᵢ^min = Eᵢ(1−dᵢ)^T` (eq 4); a budget is reachable by some allocation iff the summed floors do not exceed it (eq 6). The package provides two independent implementations that agree to the reported precision:

- `gams/BC_reachability.gms` — the model, stated as a deficit-minimising **LP feasibility program** that certifies eq 6–7 with a solver.
- `src/empirical_validation.py` — the empirical validation (Notes S5–S6): observed decline rates, the conservative bound, and the bootstrap.

```
gams/   BC_reachability.gms        the feasibility-LP model (self-contained)
src/    empirical_validation.py    Notes S5-S6 (observed rates, bound, bootstrap)
data/   data_baseline_regions.csv  nine-region baseline (E, population, income)
        backtest/                  observed national emissions + region map (15 real countries)
results/                           script outputs land here
```

Reproduction targets: summed floors **11.02 Gt**, deficit **1.02 Gt**; rate-multiplier boundary **×1.085**, horizon boundary **54.3 yr**; conservative bound **12.73 / 2.73 Gt**; fastest sustained decline **2.15 %/yr (UK)**, 9 of 15 countries declining; P(infeasible) **0.81**.

---

## Running the GAMS model

The model is self-contained — the baseline and the four allocation constructions are embedded as `TABLE`s, so no external input is needed.

```bash
gams gams/BC_reachability.gms
```

**GAMS version note.** The file is tested against GAMS 25.0.3 (the current UTM/CACEPS licence). Three compatibility adjustments vs a naive first draft are documented in the file header: `D` renamed to `Deficit` (GAMS 25 treats `D` as the derivative operator); the multiplier grid built with a `loop` rather than a direct `ord()` assignment (direct `ord()` needs GAMS 26+); and the `verdict` table built with `loop`/`if` blocks rather than `$(...)` conditionals on a two-index parameter (stricter dimension checking in 25). No logic changes; numerics are identical.

**Sets.** `i` is the nine macro-regions; `p` is the four published proposals (EPC, C&C, GDR, CER). The grid set `m` (`m1*m41`) drives the rate-multiplier sweep in §9.

**Data.** Two tables hold everything: `base(i,*)` carries current emissions `E`, population, and per-capita income `gpc`; `alloc(i,p)` carries the constructed allowances (GDR and CER are net-negative for NA and EU — transfers-required). The decline rate `d(i)` is *not* hard-coded: it is assigned from the capacity tiers against the population-weighted mean income `gbar` (Methods eq 5), so editing the baseline re-derives the frontier automatically.

**The program.** `a(i)` are the regional allowances and `D` the budget overshoot. The LP

```
min  D      s.t.   sum(i, a(i)) =l= B + D ;   a(i) =g= amin(i) ;   a(i), D =g= 0
```

returns `D* = max(0, Σ aᵢ^min − B)` — the irreducible deficit (eq 7). `D.l = 0` certifies the budget is reachable; `D.l > 0` certifies it is not. With the shipped baseline the solver returns `D.l = 1.02`, i.e. infeasible. The model status will read `1 NORMAL COMPLETION` / `1 OPTIMAL`; it is always feasible by construction (the deficit absorbs any shortfall), so a normal LP solve is expected, never `INFEASIBLE`.

**Command-line overrides.** Budget and horizon are exposed as compile-time settings:

```bash
gams gams/BC_reachability.gms --BUD=10 --HOR=54.32     # horizon boundary -> deficit ~0
gams gams/BC_reachability.gms --HOR=60                 # 60-yr horizon (Table S2)
gams gams/BC_reachability.gms lo=3 lp=cplex            # full log, choose the LP solver
```

`--BUD`/`--HOR` feed the `$if not set` / `%BUD%` mechanism at the top of the file. Any LP solver works (the program is trivial for the solver); `lp=cplex`, `lp=highs`, or the default are all fine.

**Reading the output.** Inspect the listing (`BC_reachability.lst`) for the `solve` summary and the `display` blocks: `gbar`, the derived `d(i)`, the floors `amin(i)`, `sumfloor`, `deficit`, `reachable`, the per-proposal `verdict(i,p)` and `n_clear(p)` (reproducing Fig 1c / Extended Data 1), the multiplier sweep `sf(m)` (summed floors crossing `B` near `mult = 1.085`), and the conservative bound `sumfloor_obs` / `deficit_obs` (12.73 / 2.73).

**GDX.** The model ends with

```
execute_unload 'bc_results.gdx', i, p, E, d, amin, a, D, sumfloor, deficit,
   reachable, verdict, n_clear, mult, sf, amin_obs, sumfloor_obs, deficit_obs;
```

so every symbol is persisted. Inspect or export it without re-running:

```bash
gdxdump bc_results.gdx symb=amin              # the per-region floors
gdxdump bc_results.gdx symb=sf format=csv     # the multiplier sweep, as CSV
```

To pull results into Python for figures, `gdxpds.to_dataframes('bc_results.gdx')` (or the `gams.transfer` API) reads the container directly.

**Feeding data from GDX instead of the embedded tables (optional).** If you would rather drive the model from an external baseline, replace the two `TABLE` blocks with a GDX load:

```gams
$gdxin baseline.gdx
$load i, E, pop, g, alloc
$gdxin
```

building `baseline.gdx` from the CSVs in `data/` with `csv2gdx` or a short `gams.transfer` script. The embedded tables and a GDX load give identical results; the embedded form is the default so the file runs with nothing else present.

---

## Running the empirical validation (Notes S5–S6)

```bash
pip install -r requirements.txt
python src/empirical_validation.py        # reads data/, writes results/
```

By default it reads from `data/`; override with the `BC_DATA` environment variable if your data live elsewhere. It prints the observed decline rates, the summed floors and deficit, the conservative bound, and the bootstrap P(infeasible), and writes `results/S5_observed_rates.csv` and `results/S5S6_numbers.json`.

## Data provenance

Observed national CO₂ emissions are from the **Global Carbon Project** and **Our World in Data**; income groupings follow the **World Bank** classification. The validation panel is the 15 countries with complete, reliable public records that map one-to-one onto the nine regions. The regional baseline (`data/data_baseline_regions.csv`) and the constructed allocations (embedded in the GAMS model) accompany the paper. All results reported in the Brief Communication are fully reproducible from this repository: the GAMS model, the Python stochastic runner, the input data (CSVs), and the validation scripts are all included here at **github.com/caceps/GEMS-bc**. An archived release with a persistent DOI will be minted on acceptance, and the Brief Communication will be updated to cite the tagged release.
