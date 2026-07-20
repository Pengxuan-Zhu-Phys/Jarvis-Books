# DESIGN — Configurable Feedback Return Contract (V2)

**Status**: design accepted for implementation (post-D13 hygiene; does not block D14).  
**Date**: 2026-07-20  
**Scope**: shrink and parameterize what Workers put on `hep:feedback`, while keeping
archive/DataRecorder as the sole full-observables persistence path.  
**Code targets**: `worker.py`, `worker_config.py`, `redis_queue.py`, likelihood,
sampler bases (`feedback_sampler.py`, `dynesty_sampler.py`, `mcmc_sampler.py`,
`adaptive_level_set.py`).  
**Maintainer constraint**: **D8 stays parked**. Default wire becomes **flat
`{uuid, logL}`**; extra fields are also top-level flat keys.

---

## 1. Problem

Today the Worker dual-writes after each sample:

| Path | Redis | Payload | Consumer |
|------|-------|---------|----------|
| Archive | `hep:archive_queue` | full `to_info_dict()` (params, observables, status, paths, …) | Archiver → SAMPLE + `samples.hdf5` |
| Feedback | `hep:feedback` | `{uuid, status, observables: **full nested bag**}` | Sampler control process |

Archive is correct (DataRecorder). Feedback is not:

- Nested `observables` + `status` is heavier than samplers need.
- Dynesty / MultiNest / MCMC only need **`uuid` + scalar logL**.
- Optimizers may need a few extra scalars, still as **flat** fields — not a nested bag.
- Sample lifecycle `status` belongs on archive, not on the sampler barrier.

**Goal**: feedback is a **flat science dict** owned by the active sampler policy.
Default is two keys. Extensions add sibling keys. Archive stays full and nested.

---

## 2. Principles

1. **Two channels, two jobs**
   - Archive = full persistence / DataRecorder (nested observables, status, paths).
   - Feedback = barrier inputs for the **active sampler only** (flat scalars).
2. **Default wire (every feedback method unless extended):**

   ```json
   { "uuid": "…", "logL": -12.34 }
   ```

3. **Flat only.** No nested `observables` map on feedback. Extra requested fields
   are **top-level siblings** of `uuid` / `logL`:

   ```json
   { "uuid": "…", "logL": -2.99, "delta_chi2": 3.84 }
   ```

4. **No `status` on feedback.** Persistence keeps status on archive.
5. **Always publish a finite-or-−∞ logL when `include_logl` is true.**  
   Likelihood sets **`logL = -∞`** for points that did not yield a usable
   likelihood (failed / not computed). Samplers never have to guess from “missing key”.
6. **Policy is control-owned**, stamped once into `worker_config["feedback_return"]`,
   applied by every Worker for that scan.
7. **One policy per scan** (no per-task negotiation in v1).

---

## 3. Feedback return spec

### 3.1 Wire format (published on `hep:feedback`)

**Required keys:**

| Key | Type | Notes |
|-----|------|--------|
| `uuid` | str | sample identity |
| `logL` | float | **always present** when `include_logl: true` (default). Use JSON-friendly encoding of −∞ (see §3.3). |

**Not on the wire:**

| Key | Where instead |
|-----|----------------|
| `status` | archive / DataRecorder only |
| nested `observables` | archive only; feedback never nests this bag |

**Optional extra keys (flat):** any names listed in `fields` that exist as scalars
on the sample (from params/observables after the Worker pipeline). Values are
copied as top-level keys next to `logL`.

Reserved top-level names that Workers must not overwrite from `fields`:
`uuid`, `logL` (case-sensitive on the wire as specified in §3.2).

### 3.2 Wire key spelling

| Wire key | Meaning |
|----------|---------|
| `uuid` | identity |
| `logL` | total log-likelihood (camel **L** — matches common HEP `LogL` concept, short form on the wire) |

Worker projection maps internal `observables["LogL"]` / summed `LogL*` → wire **`logL`**.  
Consumers read **`record["logL"]`** (not `observables["LogL"]`).

### 3.3 −∞ and JSON / msgpack

Likelihood / Worker must set unusable points to **`float("-inf")`** (Python) before
publish. Codecs:

- **msgpack**: native float −∞ preferred when codec is msgpack.
- **json**: encode −∞ as a sentinel string `"−inf"` / `"-inf"` (and `"+inf"` / `"nan"`
  if ever needed); `publish_feedback` / `pull_feedback` round-trip via a tiny
  encode/decode helper so consumers always see real floats after decode.

Acceptance: after `pull_feedback`, `math.isinf(record["logL"]) and record["logL"] < 0`
is true for failed points.

### 3.4 Likelihood contract (source of −∞)

**Owner:** Worker likelihood step / `LogLikelihoodEvaluator` (not the sampler).

| Outcome | Internal `observables["LogL"]` | Feedback `logL` |
|---------|--------------------------------|-----------------|
| Normal evaluation | finite float | same |
| Failed sample / exception / pass-fail soft fail that kills LogL | **`-np.inf`** | **`-inf`** |
| Selection reject (if still feedback-published) | **`-np.inf`** | **`-inf`** |

Rules:

1. Prefer a single total **`LogL`** on the sample; if only term keys `LogL_*` exist,
   sum them once when projecting (or when likelihood finalizes).
2. **Never omit `logL` on the feedback wire** when `include_logl` is true.
3. Do **not** invent fake finite placeholders (−1e300 is legacy dynesty glue only
   if a consumer still needs it internally; the **wire** value is real −∞, and
   Dynesty pool may map −∞ → engine-specific sentinel at absorb time if required).

### 3.5 Policy schema (`worker_config["feedback_return"]`)

```python
{
  "mode": "minimal" | "fields" | "all_flat",
  # minimal:   {uuid, logL}
  # fields:    {uuid, logL?, *named flat keys from sample}
  # all_flat:  {uuid, logL?, **every scalar observable as top-level key}
  #            (debug escape hatch; still flat — never nested bag)
  "fields": ["delta_chi2", "m0"],  # mode=fields only
  "include_logl": true,            # default true; false only for pure non-logL targets
}
```

When `include_logl` is false (rare ALS pure-target cards), wire may be
`{uuid, delta_chi2, …}` without `logL`. Default for all stock methods is **true**.

### 3.6 Wire examples

**Default (Dynesty / MCMC) — success:**

```json
{ "uuid": "a1b2…", "logL": -12.34 }
```

**Default — unusable point (likelihood set −∞):**

```json
{ "uuid": "a1b2…", "logL": "-inf" }
```

(decoded to `float("-inf")` on pull)

**ALS / optimizer with extra fields:**

```json
{ "uuid": "…", "logL": -2.99, "delta_chi2": 3.84 }
```

**Debug `all_flat`:** every scalar observable as a sibling key + `uuid` (+ `logL`
if included). Still no nested `observables`, no `status`.

---

## 4. Who sets the policy

### 4.1 Resolution order (first non-empty wins)

```
1. Explicit YAML: Sampling.FeedbackReturn / Sampling.feedback_return
2. Sampler.feedback_return_spec() -> dict
3. Built-in defaults by method family (§4.2)
```

### 4.2 Built-in defaults

| Method family | Default mode | fields | Rationale |
|---------------|--------------|--------|-----------|
| Dynesty, MultiNest | `minimal` | — | only uuid + logL |
| MCMC, AM, DRAM, Ensemble*, DEMCMC, PT* | `minimal` | — | only uuid + logL |
| AdaptiveLevelSet | `fields` | symbols from `target_expression` | flat target keys (+ logL) |
| Unknown feedback method | `minimal` | — | safe default |
| Stateless methods | *no feedback* | — | `publish_feedback=false` |

If ALS `target_expression` is just `LogL` / `logL`, wire collapses to the same
shape as `minimal`.

### 4.3 Sampler API

```python
class SamplingVirtual:
    def feedback_return_spec(self) -> dict:
        return {"mode": "minimal", "include_logl": True, "fields": []}
```

- Dynesty / MultiNest / MCMC: fixed `minimal`.
- ALS: `mode=fields`, `fields=target_symbols`.
- Future optimizers: declare flat field names they absorb.

### 4.4 Stamping

```python
worker_config["publish_feedback"] = …
worker_config["feedback_return"] = resolve_feedback_return(cfg, sampler)
```

---

## 5. Worker projection (implementation sketch)

```python
def build_feedback_record(
    sample: Sample,
    *,
    spec: Mapping[str, Any],
) -> dict[str, Any]:
    mode = str(spec.get("mode") or "minimal").lower()
    include_logl = bool(spec.get("include_logl", True))
    obs = dict(sample.observables or {})
    params = dict(sample.params or {})
    # Lookup bag for extra fields (observables win over params).
    bag = {**params, **obs}

    out: dict[str, Any] = {"uuid": str(sample.uuid)}

    if include_logl:
        logl = extract_logl_total(obs)  # LogL or sum LogL*; None → -inf
        out["logL"] = float(logl) if logl is not None else float("-inf")

    if mode == "minimal":
        return out

    names: list[str]
    if mode == "fields":
        names = [str(n) for n in (spec.get("fields") or [])]
    elif mode in ("all_flat", "all"):
        names = [k for k in bag.keys() if k not in ("uuid", "logL", "LogL")]
    else:
        raise ValueError(f"unknown feedback_return mode: {mode}")

    for name in names:
        if name in ("uuid", "logL"):
            continue
        if name in bag and _is_feedback_scalar(bag[name]):
            # Wire uses the requested name as-is (except LogL → already as logL).
            key = "logL" if name == "LogL" else name
            if key == "logL" and "logL" in out:
                continue
            out[key] = bag[name]
    return out

# _stage_and_submit:
self._redis.submit_result(sample.to_info_dict())   # FULL nested + status
if self._publish_feedback:
    self._redis.publish_feedback(
        build_feedback_record(sample, spec=self._feedback_return)
    )
```

`RedisQueue.publish_feedback`:

- Requires `uuid`.
- Does **not** wrap into `observables`.
- Rejects or strips accidental nested `observables` / `status` if present
  (fail-loud in debug tests; strip in production only if we prefer soft).

---

## 6. YAML surface

```yaml
Sampling:
  Method: Dynesty
  FeedbackReturn:                 # optional — Dynesty already minimal
    mode: minimal                 # minimal | fields | all_flat
    # fields: []
    # include_logl: true

  Method: AdaptiveLevelSet
  AdaptiveLevelSet:
    target_expression: "delta_chi2"
  FeedbackReturn:
    mode: fields
    fields: [delta_chi2]
    include_logl: true
```

Aliases: `feedback_return` / `FeedbackReturn`.  
Unknown `mode` → `ValueError`.  
`mode: fields` + empty `fields` + `include_logl: false` → `ValueError`.

Document in `YAML_REFERENCE_2.0.md` §6.

---

## 7. Consumer contracts

| Consumer | Today | After |
|----------|-------|--------|
| `RedisEvaluationPool` | nested `observables` + status | `record["logL"]` (float, may be −∞) |
| `MCMCSampler` | nested extract | `record["logL"]`; −∞ → reject / None policy |
| `AdaptiveLevelSet` | nested bag + status | flat `record[field]`; −∞ logL or missing target → `f=None` |
| `FeedbackSampler` failure halt | `status==Failed` | optional: halt if `logL == -inf` and policy says so; default reject |
| Archiver | full nested info | **unchanged** |

Absorb helpers **must not** read `record["status"]` or `record["observables"]`
for science. Legacy keys, if still present during a transition, are ignored once
D13.8 consumers land (single cut preferred; short dual-read only if tests force it).

Dynesty engine: if the vendored path requires a huge negative finite instead of
IEEE −∞, convert **only inside the pool** when packing `.val` for dynesty — the
Redis wire stays real −∞.

---

## 8. Non-goals

- Nested feedback bags (rejected).
- Sample `status` on feedback (rejected).
- Changing archive / DataRecorder schema.
- Per-sample dynamic field lists (v1).
- Agent / D8 surfaces.

---

## 9. Testing

| Case | Expect |
|------|--------|
| Minimal success | `{uuid, logL}` only; `logL` finite |
| Failed / no LogL | `{uuid, logL: -inf}` after decode |
| Fields mode | flat extra keys; no `observables` key |
| `all_flat` | flat scalars; no nested bag; no status |
| Archive still full | `submit_result` includes nested observables + status |
| Likelihood unit | unusable point writes `LogL = -inf` on sample |
| Dynesty pool | reads `logL`; maps −∞ for engine if needed |
| MCMC / ALS absorb | no `status` / nested `observables` dependency |
| JSON codec | −∞ round-trips through encode/decode |

---

## 10. Key Decisions

1. **Wire is flat: `{uuid, logL}` by default** — no nested `observables`.
2. **Extra fields are flat siblings**, not a second bag.
3. **Likelihood owns −∞** for uncomputed / failed points; feedback always carries
   `logL` when `include_logl` is true.
4. **No `status` on feedback** — archive only.
5. **Policy in `worker_config`**, projection on Worker; Redis transport stays dumb.
6. **Dynesty/MCMC fixed minimal**; ALS / optimizers use `fields`.
7. **Wire key `logL`** (maps from internal `LogL`).

---

## 11. Open questions

None blocking.

Optional later: warn if `fields` keys missing on many samples (monitor only).

---

## 12. Work packages / PR Plan

| WP | Title | Depends | Accept |
|----|-------|---------|--------|
| **D13.8a** | `build_feedback_record` / `resolve_feedback_return` + −∞ codec helpers; unit tests | — | flat payload shapes; −∞ round-trip |
| **D13.8b** | Likelihood / finalize: unusable → `LogL = -inf`; Worker publish flat record; drop nested bag + status from feedback | D13.8a | fakeredis body is exactly `{uuid, logL}` for Dynesty |
| **D13.8c** | Consumers: pool / MCMC / ALS / failure policy read flat `logL` + fields | D13.8b | suites green; no status/observables on absorb path |
| **D13.8d** | YAML_REFERENCE + component docs | D13.8b | docs match flat contract |

**Rollback**: temporary dual-read of legacy nested feedback in consumers only if a
mid-migration hotfix is required; goal is single-cut flat wire.

---

## 13. Relation to existing docs

- Replaces earlier draft wording that kept nested `observables` on feedback.
- Updates absorb notes in [`components/feedback_sampler.md`](components/feedback_sampler.md)
  and feedback wording in [`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md).
- Independent of D14 (same stamped `worker_config` on remote Workers).
