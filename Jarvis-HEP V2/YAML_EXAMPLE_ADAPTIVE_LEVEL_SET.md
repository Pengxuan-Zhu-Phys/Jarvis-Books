# YAML Example — AdaptiveLevelSet

Minimal task-YAML skeleton for the **AdaptiveLevelSet** sampler
(`Method: AdaptiveLevelSet`). Top-level blocks mirror a normal scan; nested
options are **placeholders only** — fill them from
[`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) and
[`components/adaptive_voronoi_contour.md`](components/adaptive_voronoi_contour.md).

```yaml
project_name: adaptive-level-set-example

Scan:
  name: levelset-01
  # …

Runtime:
  mode: redis
  workers: 4
  # …

Sampling:
  Method: AdaptiveLevelSet
  Seed: 42
  Variables:
    - name: x
      # distribution: …
    - name: y
      # distribution: …
    # … (2–5 variables)
  AdaptiveLevelSet:
    # target_expression / target_value / …
    # …

Mapper:
  # …

LibDeps:
  # …

Calculators:
  # Modules: …

Operas:
  # Modules: …

Likelihood:
  # expressions: …
```

**Notes**

- `Sampling.Method: AdaptiveLevelSet` selects the feedback-driven level-set sampler.
- Required AdaptiveLevelSet fields at runtime: `target_expression`, `target_value`
  (see component design). Other knobs stay optional with code defaults.
- Workers auto-enable `publish_feedback` for this method; no extra YAML flag.
- Output: ordinary DATABASE/SAMPLE plus `levelset.json` under `task_result_dir`.
