# Component — Utilities, versioning, convert & plot (`jarvishep2/utils.py`, `versioning.py`, `dataconvert.py`, `plot.py`)

**Role**: the small shared helpers — duration/format, interpolation builders, HDF5→CSV
conversion, plotting, and version/banner branding. Grouped because each is thin and mostly
reused from V1.
**Status**: design — auxiliary; reused with light retargeting to `Jarvis2`.
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11 (reused);
[datarecorder.md](datarecorder.md) (CSV format is the Archiver's frozen contract).
**Reuses V1**: `utils.py`, `versioning.py`, `dataconvert.py`, `plot.py`, `observable_io.py`
(CSV fieldnames).

---

## 1. Responsibilities

1. **utils**: `format_duration`, interpolation function builders
   (`get_interpolate_1D_function_from_config`), misc small helpers used in logs/expressions.
2. **versioning**: `Jarvis2` banner/logo, `-v`, `--refs` text, single `\Jversion`-style source.
3. **dataconvert**: HDF5 → CSV (`--convert`) using the **frozen** `observable_io` field policy.
4. **plot**: plotting engine for `--plot` (config generation + render), unchanged behavior.

These are deliberately **not** on the per-Sample hot path (except `format_duration` in logs and
interpolation funcs registered into the [expression engine](expression.md)).

---

## 2. Member functions (selected)

| Function | Module | Behavior |
|----------|--------|----------|
| `format_duration` | utils | seconds → human string (logs, run_summary, factory status). |
| `get_interpolate_1D_function_from_config` | utils | build a callable interpolator from config → registered into the expression context (`update_funcs`). |
| `convert_hdf5_to_csv` | dataconvert/utils | stream HDF5 records → CSV via `observable_io` field policy (frozen format). |
| `print_banner` / `version` / `refs` | versioning | `Jarvis2` branding (distinct logo from `Jarvis`), version string, full reference. |
| `make_plot` / `plot_config` | plot | `--plot` config + render. |

---

## 3. Convert & plot flow

```
Jarvis2 <task>.yaml --convert  → dataconvert.convert_hdf5_to_csv(DATABASE/*.hdf5)
                                   → CSV via observable_io.flatten_records_for_csv (frozen)
Jarvis2 <task>.yaml --plot     → plot.make_plot(config)
```

CSV output is **byte-identical to V1** for the same DATABASE (the conversion is the same
`observable_io` policy; the Archiver only changed *how* records were written, not the schema).

---

## 4. Lifecycle / failure semantics

- All control-process, one-shot or log-time helpers — no Redis, no Workers.
- `convert` is **read-only** over a finished DATABASE; safe to run alongside or after a scan.
- Versioning reads a single source-of-truth version constant (no hard-pinned strings scattered).
- Plot failures (missing optional deps) degrade with a clear message, never crash a scan.

---

## 5. Interfaces

- **utils** → logs/run_summary (`format_duration`), expression engine (interpolators).
- **dataconvert** → `observable_io` (frozen CSV schema); CLI `--convert`.
- **versioning** → CLI banner / `-v` / `--refs`.
- **plot** → CLI `--plot`.

---

## 6. Tests (`tests/test_utils_convert_plot.py`)

Unit:
1. **format_duration** — boundary values (0, sub-second, hours) format as expected.
2. **Interpolator** — built callable matches the config nodes; registers into an expression and
   evaluates.
3. **CSV parity** — `convert_hdf5_to_csv` on a fixed DATABASE is **byte-identical** to the V1
   golden CSV (frozen `observable_io` policy).
4. **Versioning** — `-v`/`--refs` print the single-source version; banner is the `Jarvis2`
   variant (distinct from `Jarvis`).
5. **Plot smoke** — `make_plot` on a fixture produces an output file (or a clear skip if optional
   deps absent).

Verification logic: test 3 is the frozen-output gate for `--convert`; the rest are cheap
correctness checks on reused helpers.

---

## 7. Open questions

- Whether `--convert` should also run automatically at end-of-scan (currently explicit; keep
  explicit to match V1).
- Plot engine modernization (out of scope for the runtime rebuild; track separately).
