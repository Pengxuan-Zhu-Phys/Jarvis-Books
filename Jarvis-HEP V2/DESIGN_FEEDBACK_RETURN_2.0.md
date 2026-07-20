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
| Archive | `hep:archive_queue` | full `to_info_dict()` (params, observables, **status**, paths, …) | Archiver → SAMPLE + `samples.hdf5` |
| Feedback | `hep:feedback` | `{uuid, status, observables: **full**}` | Sampler control process |

Archive is correct: it is the DataRecorder path. Feedback is not:

- Comments call it a “light” record, but it copies **every** observable **and** `status`.
- Dynesty / MultiNest / MCMC only need **`uuid` + `LogL`**.
- AdaptiveLevelSet (and future optimizers) need **selected** target fields, not the whole bag.
- **Sample `status` is for persistence / ops, not for sampler science** — the control
  process should not depend on Worker lifecycle status to accept/reject a proposal.
- Full observables on the barrier channel waste bandwidth and hide the real contract.

**Goal**: make feedback **sampler-owned and declarative**. Default minimal; opt-in for
arbitrary keys; stamp the policy into Worker config at scan start so Workers project
observables before `rpush`. **Do not put `status` on the feedback wire.**

---

## 2. Principles

1. **Two channels, two jobs**
   - Archive = truth for persistence / analysis / DataRecorder (includes `status`).
   - Feedback = barrier inputs for the **active sampler only** (science scalars only).
2. **Default = minimal** for every feedback-driven sampler:  
   `{uuid, observables: {LogL}}` (LogL omitted when unavailable / failed).
3. **No `status` on feedback.** Sampler failure is inferred from **missing science
   values** (no `LogL`, missing target field), not from sample lifecycle status.
4. **Extension is first-class**, not a hack: samplers and YAML may request extra
   observable names (or `all` for debug).
5. **Policy is decided on the control process**, stamped once into `worker_config`,
   applied identically by every Worker for that scan.
6. **Wire format**: top-level **`uuid` + `observables`** only (`observables` is a
   projected map). Drop `status` from the published feedback payload.
7. **No per-task negotiation** in v1 of this design: one policy per Worker pool /
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

**Always present on the wire:**

| Key | Type | Notes |
|-----|------|--------|
| `uuid` | str | required |

**Not on the feedback wire:**

| Key | Where it lives instead |
|-----|------------------------|
| `status` | Archive / DataRecorder only (`submit_result` / `to_info_dict`) |

**Inside `observables` after projection:**

| Mode | Contents |
|------|----------|
| `minimal` | `{LogL: float}` when available (and `include_logl`) |
| `fields` | requested names that exist on the sample, plus LogL if `include_logl` |
| `all` | shallow copy of full `sample.observables` (still no top-level `status`) |

Missing requested fields are **omitted** (not null-filled). Consumers must tolerate
absence.

### 3.2 LogL extraction rule (Worker projection)

Same defensive rule control already uses when reading:

1. If `observables["LogL"]` exists → use it.
2. Else if any keys match `LogL*` (excluding the total) → sum them into a single
   projected `LogL` for the feedback map.
3. Else → omit `LogL`.

**Failed / unusable samples:** do **not** invent a fake LogL. Project an empty or
partial `observables` map (typically `{}` under `minimal`). The sampler treats
**missing LogL** (or missing required target field) as reject / −∞ / `f=None`.

If the sample completed physics but Likelihood failed to write LogL, the same
“missing LogL” path applies — which is what the sampler needs for its barrier,
independent of whether archive records `status: Failed` or `Completed`.

### 3.3 Failure semantics (replacing `status` on feedback)

| Situation | Feedback body | Sampler absorb |
|-----------|---------------|----------------|
| Success with LogL | `{uuid, observables: {LogL: x, …}}` | use LogL / target fields |
| Failed or no LogL | `{uuid, observables: {}}` or without `LogL` | reject / −∞ / `f=None` |
| Success, non-LogL target only | `{uuid, observables: {delta_chi2: …}}` | ALS uses target keys |

Implementation note: existing code that branches on `record["status"] == "Failed"`
must be updated to **value presence** checks (`LogL` / target keys). Archive still
carries real status for humans and DataRecorder.

### 3.4 Wire example

**Default (Dynesty / MCMC):**

```json
{
  "uuid": "a1b2…",
  "observables": { "LogL": -12.34 }
}
```

**Failed / missing LogL:**

```json
{
  "uuid": "a1b2…",
  "observables": {}
}
```

**ALS with target fields:**

```json
{
  "uuid": "…",
  "observables": {
    "LogL": -2.99,
    "delta_chi2": 3.84
  }
}
```

**Debug / `mode: all`:** full observables map under `observables`, still **no**
top-level `status`.

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
self._redis.submit_result(info)          # FULL (incl. status) — unchanged
if self._publish_feedback:
    self._redis.publish_feedback({
        "uuid": sample.uuid,
        # no status
        "observables": project_feedback_observables(
            sample.observables,
            spec=self._feedback_return,
        ),
    })
```

`RedisQueue.publish_feedback` stays a thin transport (validate `uuid` only);
**projection lives in the Worker** so the archive path never accidentally shrinks.
Optional: strip any accidental `status` key if a caller passes one.

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

## 7. Consumer contracts

| Consumer | Today | After this design |
|----------|-------|-------------------|
| `RedisEvaluationPool` | `status`→−1e300; else `LogL` | **missing LogL** → −1e300 / fail path; else `LogL` |
| `MCMCSampler._extract_logl` | `status==Failed` → None | **missing LogL** → None (reject) |
| `FeedbackSampler._failure_policy_halt` | `status==Failed` | redefine on missing science values, or drop halt-on-status; archive remains source of Failed counts |
| `AdaptiveLevelSet.absorb` | `status` + target keys | **missing target / LogL** → `f=None` |
| Archiver / DATABASE | full info incl. status | **unchanged** |

Absorb helpers should not read `record.get("status")` for science decisions.

---

## 8. Non-goals

- Changing archive / DataRecorder schema (status stays there).
- Streaming intermediate calculator observables mid-workflow onto feedback.
- Per-sample dynamic field lists (v1).
- Compressing archive payloads.
- Agent / D8 surfaces.
- Putting sample lifecycle status back on the feedback channel for “convenience”.

---

## 9. Testing

| Case | Expect |
|------|--------|
| Unit: `project_feedback_observables` minimal / fields / all | map shapes; no status key |
| Unit: LogL sum fallback when only `LogL_a`/`LogL_b` | single `LogL` in projection |
| Worker + fakeredis: Dynesty pool drain | feedback JSON keys ⊆ `{uuid, observables}`; observables ⊆ `{LogL}` |
| Worker: Failed sample | feedback has uuid, empty/no LogL; archive still has status Failed |
| Worker: ALS fields mode | target key present; unrelated bulky keys absent |
| Archive side: `submit_result` still full | Archiver tests unchanged |
| YAML `mode: all` | full observables under `observables`, still no top-level status |
| Default resolution without YAML | Dynesty → minimal; ALS → fields(symbols) |
| Consumer unit: MCMC/Dynesty/ALS | no dependency on feedback `status` |

---

## 10. Key Decisions

1. **Default minimal (`uuid` + `LogL`)** for all feedback methods unless they declare
   otherwise — matches Dynesty/MCMC reality and cuts barrier traffic.
2. **`status` is not part of feedback.** Persistence keeps status on archive;
   samplers use missing LogL / missing target fields as the failure signal.
3. **Keep `observables` key** on the wire (projected map) — extension for optimizers
   without inventing a second field bag; only the map content shrinks.
4. **Policy stamped in `worker_config`**, not per-task — one scan, one contract;
   Workers stay simple.
5. **Projection in Worker, not RedisQueue** — archive path cannot be accidentally
   filtered; transport stays dumb.
6. **YAML override + sampler defaults** — optimizers keep an open interface;
   stock nested/MCMC cards need no new keys.
7. **ALS auto-fields from `target_expression`** — preserves current ALS cards without
   forcing users to list FeedbackReturn.

---

## 11. Open questions

None blocking. Optional later:

- Should `mode: fields` warn when a requested key is missing on >N% of samples?
  (nice-to-have monitor, not required for v1.)
- Rename wire field to `feedback` later? Rejected for now — keep `observables`.

---

## 12. Work packages / PR Plan

| WP | Title | Depends | Accept |
|----|-------|---------|--------|
| **D13.8a** | Spec helpers: `project_feedback_observables`, `resolve_feedback_return`, unit tests | — | pure functions green; no status in payload |
| **D13.8b** | Worker applies `worker_config["feedback_return"]`; stamp from `build_worker_config`; drop status from `publish_feedback` | D13.8a | Dynesty feedback body = uuid + LogL only under fakeredis |
| **D13.8c** | Consumer cleanup: MCMC / pool / ALS / failure policy use missing values not status; sampler defaults (Dynesty/MCMC minimal; ALS fields) | D13.8b | ALS e2e still fills target `f`; Failed → reject without status |
| **D13.8d** | YAML_REFERENCE + feedback_sampler / redis_queue / worker component docs | D13.8b | docs match wire contract |

**Rollback**: set `FeedbackReturn.mode: all` for observables map size only — status
still stays off the feedback channel once D13.8b lands.

---

## 13. Relation to existing docs

- Supersedes the informal “light record = full observables + status” wording in
  [`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md) §5
  and [`components/feedback_sampler.md`](components/feedback_sampler.md) §3 absorb note.
- Complements D13.7b (unmatched feedback logging): smaller payloads, same drain.
- Independent of D14 cluster (same `worker_config` template will carry the policy).
