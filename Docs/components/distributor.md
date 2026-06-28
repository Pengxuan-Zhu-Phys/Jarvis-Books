# Component — Distributor (sampler dispatch + resume registry) (`jarvishep2/distributor.py`)

**Role**: select the sampler implementation by name and own the **resume-support registry** (which
samplers can resume). The single place that maps a YAML `Sampling.Method` string to a concrete
sampler class.
**Status**: design — reused, near-unchanged (the dispatch table grows new V2 samplers only if any
are added; none are planned).
**Design refs**: [`../DESIGN_2.0_DISTRIBUTED.md`](../DESIGN_2.0_DISTRIBUTED.md) §11;
[sampler.md](sampler.md), [samplers_catalog.md](samplers_catalog.md), [cli.md](cli.md).
**Reuses V1**: `Distributor(Base)` (`jarvishep/distributor.py`) — `set_method` (match/case),
`RESUME_SUPPORT_STATUS`, `get_resume_status`. Kept as-is; only `set_method` returns a sampler that
submits through Redis (the change lives in the [sampler base](sampler.md), not here).

---

## 1. Responsibilities

1. **Dispatch**: `Sampling.Method` name → instantiate the matching sampler class (lazy import per
   branch, so unused sampler deps aren't loaded).
2. **Resume registry**: declare each sampler's resume support (`implemented` /
   `intentionally unsupported`); drive the `--resume` prompt and `--refs` output.
3. Keep aliases working: `ToyMCMC`→`MCMC`, `DREAMLite`/`DREAM-lite`, `PTMCMC`→`tpmcmc`, etc.

It is a **pure selector + registry** — no execution, no state.

---

## 2. Structure (reused)

```python
class Distributor(Base):
    RESUME_SUPPORT_STATUS: dict[str, str]      # {"MCMC": "implemented", ...}
    _VALID_RESUME_STATUSES = {"implemented", "intentionally unsupported"}

    @classmethod
    def get_resume_status(cls, method: str) -> str: ...
    @classmethod
    def list_resume_statuses(cls) -> dict: ...
    @staticmethod
    def set_method(method: str): ...           # match/case → sampler instance (lazy import)
```

---

## 3. Member functions

| Method | Signature | Behavior |
|--------|-----------|----------|
| `set_method` | `(method) -> SamplingVirtial` | `match`/`case` on the name; lazy-import + instantiate the sampler; raise on unknown method. |
| `get_resume_status` | `(method) -> str` | `implemented` / `intentionally unsupported` for the `--resume` flow. |
| `list_resume_statuses` | `() -> dict` | Full registry (for `--refs` / docs). |

---

## 4. V2 notes

- The **only** V2 change is that the sampler returned by `set_method` submits through
  `redis.push_task` (via the [sampler base](sampler.md)) instead of `factory.submit_task`. The
  Distributor's table and logic are unchanged.
- Lazy per-branch imports keep optional sampler dependencies (Dynesty, MultiNest, etc.) out of the
  import graph unless selected — important when V2 ships a leaner core.
- The resume registry is consumed at boot to decide whether `--resume` is offered for the chosen
  sampler.

---

## 5. Concurrency / lifecycle / failure semantics

- Control-process only, one-shot at boot (sampler selection).
- Unknown method → clear error at boot (not after setup).
- No shared state, trivially safe.

---

## 6. Interfaces

- **core.init_sampler** → `Distributor.set_method(config.Sampling.Method)`.
- **resume / checkpoint** → `get_resume_status` gates the `--resume` prompt.
- **cli / `--refs`** → `list_resume_statuses`.

---

## 7. Tests (`tests/test_distributor.py`)

Unit:
1. **Dispatch coverage** — every name in `RESUME_SUPPORT_STATUS` resolves via `set_method` to the
   right class (incl. aliases `ToyMCMC`, `DREAMLite`).
2. **Unknown method** — raises a clear error.
3. **Lazy import** — selecting `Random` does not import Dynesty/MultiNest (assert on `sys.modules`).
4. **Resume registry** — `get_resume_status` returns the declared value; `--resume` prompt logic
   honors it.

Verification logic: test 1 keeps the full sampler surface reachable; test 3 protects V2's lean
import graph.

---

## 8. Open questions

- Whether to data-drive the dispatch table (registry decorator) instead of a `match`/`case`
  (cosmetic; keep `match`/`case` to match V1 and stay explicit).
