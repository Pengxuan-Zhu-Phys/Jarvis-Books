# DESIGN — Configurable Feedback Return Contract (V2)

**Status**: design accepted for implementation (post-D13 hygiene; does not block D14).  
**Date**: 2026-07-20  
**Scope**: shrink and parameterize what Workers put on `hep:feedback`, while keeping
archive/DataRecorder as the sole full-observables persistence path.  
**Code targets**: `worker.py`, `worker_config.py`, `redis_queue.py`, sampler bases
(`feedback_sampler.py`, `dynesty_sampler.py`, `mcmc_sampler.py`, `adaptive_level_set.py`).  
**Maintainer constraint**: **D8 stays parked**. Additive only — existing YAML cards
keep running; default becomes **minimal**.

---

## 1. Problem

Today the Worker dual-writes after each sample:

| Path | Redis | Payload | Consumer |
|------|-------|---------|----------|
| Archive | `hep:archive_queue` | full `to_info_dict()` (params, observables, paths, …) | Archiver → SAMPLE + `samples.hdf5` |
| Feedback | `hep:feedback` | `{uuid, status, observables: **full**}` | Sampler control process |

Archive is correct: it is the DataRecorder path. Feedback is not:

- Comments call it a “light” record, but it copies **every** observable.
- Dynesty / MultiNest / MCMC only need **`uuid` + `LogL`** (+ `status`).
- AdaptiveLevelSet (and future optimizers) need **selected** target fields, not the whole bag.
- Full observables on the barrier channel waste bandwidth and hide the real contract.

**Goal**: make feedback **sampler-owned and declarative**. Default minimal; opt-in for
arbitrary keys; stamp the policy into Worker config at scan start so Workers project
observables before `rpush`.

---

## 2. Principles

1. **Two channels, two jobs**
   - Archive = truth for persistence / analysis / DataRecorder.
   - Feedback = barrier inputs for the **active sampler only**.
2. **Default = minimal** for every feedback-driven sampler:  
   `{uuid, status, LogL}` (LogL may be absent on Failed).
3. **Extension is first-class**, not a hack: samplers and YAML may request extra
   observable names (or `all` for debug).
4. **Policy is decided on the control process**, stamped once into `worker_config`,
   applied identically by every Worker for that scan.
5. **Wire format stays compatible**: keep top-level keys `uuid`, `status`,
   `observables` (a **projected** map). Consumers already read that shape; only the
   map shrinks.
6. **No per-task negotiation** in v1 of this design: one policy per Worker pool /
   scan. (Per-task override is a non-goal until a sampler proves it needs it.)

---

## 3. Feedback return spec

### 3.1 Schema (pickled into `worker_config["feedback_return"]`)

```python
{
  "mode": "minimal" | "fields" | "all",
  # mode=minimal: observables ⊆ {LogL} (see §3.2)
  # mode=fields:  observables = {LogL?} ∪ requested keys present on sample
  # mode=all:     observables = full sample.observables (escape hatch / debug)
  "fields": ["delta_chi2", "m0"],   # only used when mode == "fields"
  "include_logl": true,             # default true; false only for pure non-LogL targets
}
```

**Always present on the wire (outside `observables`):**

| Key | Type | Notes |
|-----|------|--------|
| `uuid` | str | required |
| `status` | str | `Completed` / `Failed` / … |

**Inside `observables` after projection:**

| Mode | Contents |
|------|----------|
| `minimal` | `{LogL: float}` when available (and `include_logl`) |
| `fields` | requested names that exist on the sample, plus LogL if `include_logl` |
| `all` | shallow copy of full `sample.observables` |

Missing requested fields are **omitted** (not null-filled). Consumers must tolerate
absence (Failed samples already do).

### 3.2 LogL extraction rule (Worker projection)

Same defensive rule control already uses when reading:

1. If `observables["LogL"]` exists → use it.
2. Else if any keys match `LogL*` (excluding the total) → sum them into a single
   projected `LogL` for the feedback map.
3. Else → omit `LogL` (MCMC/Dynesty treat as failure / −inf path).

Projection **does not invent** fake LogL for Failed status; Failed records may omit
LogL entirely.

### 3.3 Wire example

**Default (Dynesty / MCMC):**

```json
{
  "uuid": "a1b2…",
  "status": "Completed",
  "observables": { "LogL": -12.34 }
}
```

**ALS with target fields:**

```json
{
  "uuid": "…",
  "status": "Completed",
  "observables": {
    "LogL": -2.99,
    "delta_chi2": 3.84
  }
}
```

**Debug / `mode: all`:** current behavior (full map).

---

## 4. Who sets the policy

### 4.1 Resolution order (first non-empty wins)

```
1. Explicit YAML: Sampling.FeedbackReturn / Sampling.feedback_return
2. Sampler class/method override: Sampler.feedback_return_spec(config) -> dict
3. Built-in defaults by method family (§4.2)
```

YAML always wins so a user can force `all` for debugging without code changes.

### 4.2 Built-in defaults (when YAML absent)

| Method family | Default mode | fields | Rationale |
|---------------|--------------|--------|-----------|
| Dynesty, MultiNest | `minimal` | — | Nested sampling only needs logL |
| MCMC, AM, DRAM, Ensemble*, DEMCMC, PT* | `minimal` | — | Metropolis only needs logL |
| AdaptiveLevelSet | `fields` | symbols from `target_expression` (+ LogL if present) | Target eval needs those keys |
| Unknown feedback method | `minimal` | — | Safe bandwidth default |
| Stateless methods | *no feedback* | — | `publish_feedback=false` unchanged |

ALS field discovery: parse `target_expression` free symbols that are not numeric
literals; intersect with “likely observables” is **not** required — request the
symbol names; Worker omits missing ones. If expression is just `LogL`, ALS
collapses to the same wire shape as `minimal`.

### 4.3 Sampler API

```python
class SamplingVirtual:
    def feedback_return_spec(self) -> dict:
        """Return worker_config['feedback_return'] fragment for this method."""
        return {"mode": "minimal", "include_logl": True, "fields": []}
```

Overrides:

- `DynestySampler` / `MultiNestSampler` / `MCMCSampler`: keep `minimal` (fixed).
- `AdaptiveLevelSetSampler`: `mode=fields`, `fields=target_symbols`.
- Future optimizers (HMC control, RL, …): declare what they absorb.

### 4.4 Stamping into Workers

In `build_worker_config` / Core scan setup (same place as `publish_feedback`):

```python
worker_config["publish_feedback"] = …
worker_config["feedback_return"] = resolve_feedback_return(cfg, sampler)
```

Workers read the dict once at `__init__` and reuse for every sample. No Redis
round-trip for the policy itself.

---

## 5. Worker projection (implementation sketch)

```python
def project_feedback_observables(
    observables: Mapping[str, Any],
    *,
    spec: Mapping[str, Any],
    status: str,
) -> dict[str, Any]:
    mode = str(spec.get("mode") or "minimal").lower()
    include_logl = bool(spec.get("include_logl", True))
    if mode == "all":
        return dict(observables)
    out: dict[str, Any] = {}
    if mode == "fields":
        for name in spec.get("fields") or []:
            key = str(name)
            if key in observables:
                out[key] = observables[key]
    if include_logl and "LogL" not in out:
        logl = extract_logl_total(observables)  # LogL or sum LogL*
        if logl is not None:
            out["LogL"] = logl
    return out

# _stage_and_submit:
info = sample.to_info_dict()
self._redis.submit_result(info)          # FULL — unchanged
if self._publish_feedback:
    self._redis.publish_feedback({
        "uuid": sample.uuid,
        "status": sample.status,
        "observables": project_feedback_observables(
            sample.observables,
            spec=self._feedback_return,
            status=sample.status,
        ),
    })
```

`RedisQueue.publish_feedback` stays a thin transport; **projection lives in the
Worker** so the archive path never accidentally shrinks.

---

## 6. YAML surface

### 6.1 Optional block

```yaml
Sampling:
  Method: Dynesty
  # Optional — Dynesty already defaults to minimal
  FeedbackReturn:
    mode: minimal          # minimal | fields | all
    # fields: []           # only for mode: fields
    # include_logl: true

  Method: AdaptiveLevelSet   # example override
  AdaptiveLevelSet:
    target_expression: "delta_chi2"
  FeedbackReturn:
    mode: fields
    fields: [delta_chi2, LogL]
```

### 6.2 Alias

Accept `feedback_return` (snake_case) as well as `FeedbackReturn`.

### 6.3 Validation

- Unknown `mode` → `ValueError` at config load (fail loud).
- `mode: fields` with empty `fields` and `include_logl: false` → `ValueError`
  (empty projection is useless for a feedback sampler).
- Stateless methods: block ignored (no publish).

Document in `YAML_REFERENCE_2.0.md` §6 (Sampling).

---

## 7. Consumer contracts (no change required if defaults correct)

| Consumer | Reads | After this design |
|----------|-------|-------------------|
| `RedisEvaluationPool` | `uuid`, `LogL` via extract | works with `minimal` |
| `MCMCSampler._extract_logl` | `status`, `LogL` / `LogL*` | works with `minimal` |
| `AdaptiveLevelSet.absorb` | target keys in observables | needs `fields` or `all` |
| Archiver / DATABASE | archive payload only | **unchanged** |

If ALS is misconfigured as `minimal` while `target_expression` is not `LogL`,
target eval returns `None` / fails conservatively — same as missing keys today.
Prefer auto-fields from the expression (§4.2) so the default ALS card keeps working.

---

## 8. Non-goals

- Changing archive / DataRecorder schema.
- Streaming intermediate calculator observables mid-workflow onto feedback.
- Per-sample dynamic field lists (v1).
- Compressing archive payloads.
- Agent / D8 surfaces.

---

## 9. Testing

| Case | Expect |
|------|--------|
| Unit: `project_feedback_observables` minimal / fields / all | map shapes |
| Unit: LogL sum fallback when only `LogL_a`/`LogL_b` | single `LogL` in projection |
| Worker + fakeredis: Dynesty pool drain | feedback JSON has only uuid/status/LogL keys under observables |
| Worker: ALS fields mode | target key present; unrelated bulky keys absent |
| Archive side: `submit_result` still full | Archiver tests unchanged |
| YAML `mode: all` | parity with pre-change feedback body |
| Default resolution without YAML | Dynesty → minimal; ALS → fields(symbols) |

---

## 10. Key Decisions

1. **Default minimal (`uuid`+`status`+`LogL`)** for all feedback methods unless they
   declare otherwise — matches Dynesty/MCMC reality and cuts barrier traffic.
2. **Keep `observables` key** on the wire (projected map) — zero churn for absorb
   helpers; only the map content shrinks.
3. **Policy stamped in `worker_config`**, not per-task — one scan, one contract;
   Workers stay simple.
4. **Projection in Worker, not RedisQueue** — archive path cannot be accidentally
   filtered; transport stays dumb.
5. **YAML override + sampler defaults** — optimizers keep an open interface;
   stock nested/MCMC cards need no new keys.
6. **ALS auto-fields from `target_expression`** — preserves current ALS cards without
   forcing users to list FeedbackReturn.

---

## 11. Open questions

None blocking. Optional later:

- Should `mode: fields` warn when a requested key is missing on >N% of Completed
  samples? (nice-to-have monitor, not required for v1.)
- Rename wire field to `feedback` later? Rejected for now — keep `observables`.

---

## 12. Work packages / PR Plan

| WP | Title | Depends | Accept |
|----|-------|---------|--------|
| **D13.8a** | Spec helpers: `project_feedback_observables`, `resolve_feedback_return`, unit tests | — | pure functions green |
| **D13.8b** | Worker applies `worker_config["feedback_return"]`; stamp from `build_worker_config` | D13.8a | Dynesty feedback body minimal under fakeredis |
| **D13.8c** | Sampler defaults: Dynesty/MCMC minimal; ALS fields(symbols) | D13.8b | ALS e2e still fills target `f` |
| **D13.8d** | YAML_REFERENCE + feedback_sampler / redis_queue / worker component docs | D13.8b | docs match wire contract |

**Rollback**: set `FeedbackReturn.mode: all` globally in YAML or default mode temporarily
to restore pre-change payloads.

---

## 13. Relation to existing docs

- Supersedes the informal “light record = full observables” wording in
  [`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md) §5
  and [`components/feedback_sampler.md`](components/feedback_sampler.md) §3 absorb note.
- Complements D13.7b (unmatched feedback logging): smaller payloads, same drain.
- Independent of D14 cluster (same `worker_config` template will carry the policy).
