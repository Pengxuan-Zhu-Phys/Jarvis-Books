# Jarvis-HEP V2 ↔ Jarvis-Agent Bridge — Agent API & Native Tool Support

**Version**: 1.0 (2026-07-03)
**Status**: Approved design — authoritative for the V2-side Agent API surface
**Owners**: V2 side = Jarvis-HEP-v2 (`jarvishep2`); Agent side = Jarvis-Agent
**Companion docs**:
[`DESIGN_2.0_DISTRIBUTED.md`](DESIGN_2.0_DISTRIBUTED.md) (V2 architecture),
[`YAML_REFERENCE_2.0.md`](YAML_REFERENCE_2.0.md) (task-YAML as-built),
[`V2_DISTRIBUTED_PLAN.md`](V2_DISTRIBUTED_PLAN.md) §D8 (V2 work packages),
`Jarvis-Agent/docs/DESIGN.md` §7 (tool system) and
`Jarvis-Agent/docs/HEP_RUNTIME_TOOLS.md` (agent-side tool family spec).

> **What this document decides.** Jarvis-Agent gets first-class, native tools to drive
> Jarvis-HEP V2 scans (validate → smoke → start → monitor → results → stop). To make those
> tools reliable, V2 exposes a small, versioned, machine-readable control surface — the
> **Agent API** — consisting of additive CLI verbs with JSON output, one run-state file
> contract, and a graceful-stop signal contract. This document is the single source of truth
> for that API. The agent-side tool ergonomics (schemas, tiers, digests) are specified in
> `HEP_RUNTIME_TOOLS.md` and must not redefine the wire contract.

---

## 1. Motivation & Constraints

Jarvis-Agent's pipeline (recon → deep-parse → generate → validate, DESIGN.md §8) ends today at
*validation* — `jarvis_hep_dryrun` is its only V2-facing tool, and the actual scan lifecycle
(submit, watch, harvest, stop, resume) has no tool support. Meanwhile V2's user surface is
human-oriented: `Jarvis2Core.run()` blocks until the scan completes, `--monitor` prints a
formatted text table, results live in HDF5, and nothing emits machine-readable status.

Binding constraints inherited from both designs:

1. **Option A separation** (Agent design decision, 2026-06-27): V2 stays a pure execution
   engine; the agent never imports `jarvishep2` in-process. Integration = CLI subprocess +
   filesystem + read-only Redis.
2. **Agent tool discipline** (Agent DESIGN.md §7): tools are few and rigid, errors are data,
   every external command goes through the ExecutionGateway and lands in the journal.
3. **V2 frozen contracts** (DESIGN_2.0 §0): existing CLI flags, output layout, run_summary,
   and checkpoint UX stay frozen — everything added here is **additive**.
4. **Local-first**: both projects run on the same machine (or share a filesystem + Redis);
   no network service is introduced by this bridge.

## 2. Decision Record

| # | Decision | Rationale |
|---|----------|-----------|
| B1 | Transport = **CLI verbs with `--json` + state files + read-only Redis**; no daemon, no RPC server, no MCP (yet) | Process boundary (Option A) with zero new long-lived services; gateway/journal compatible; MCP noted as future re-entry (Agent DESIGN §10.2) |
| B2 | All machine output uses **one JSON envelope** with `api_version` (§4.1) | Version handshake + uniform agent-side parsing |
| B3 | The control process writes **`run_state.json`** in the scan output dir (§5) | Status must survive process exit and work without Redis access |
| B4 | **Graceful stop = SIGINT/SIGTERM contract** on the control PID (§6) | Reuses the existing checkpoint machinery; no new IPC |
| B5 | Scan starting stays **agent-side** (gateway-managed background `Jarvis2 <yaml>`); V2 does not grow a `--daemon` mode | The agent owns process supervision + journaling; V2 stays simple |
| B6 | `--validate` becomes a real V2 verb (config load + normalization + diagnostics, no runtime bring-up) | The agent's Phase D needs "real Jarvis-HEP as final judge" without paying for a run; also fixes silent-coercion review findings (YAML_REFERENCE §13, A.10) |
| B7 | Agent-facing tool family = **6 tools** `hep_validate / hep_smoke / hep_scan_start / hep_scan_status / hep_scan_stop / hep_results` | Fits the ≤18-tool budget; realizes and supersedes the reserved `jarvis_hep_dryrun` slot |

## 3. Architecture Overview

```
 Jarvis-Agent (AgentLoop + ToolRegistry)
 │
 │  hep_validate ──────► Jarvis2 <yaml> --validate --json          (one-shot, read-only)
 │  hep_smoke ─────────► Jarvis2 <yaml> --check-modules --json     (one-shot, sandboxed run)
 │  hep_scan_start ────► Jarvis2 <yaml> [--resume]                 (gateway background process)
 │  hep_scan_status ───► read  outputs/<scan>/run_state.json       (always)
 │                       + Jarvis2 --monitor-json                  (live enrich, optional)
 │  hep_results ───────► Jarvis2 <yaml> --results --json ...       (bounded rows + stats)
 │  hep_scan_stop ─────► SIGINT → control PID (from run_state)     (checkpoint + exit)
 │
 └── all subprocess calls via ExecutionGateway (fixed argv, journal, timeout, artifact capture)

 Jarvis-HEP V2 control process
 ├── jarvishep2/api.py        NEW: envelope, validate/results/status implementations
 ├── jarvishep2/run_state.py  NEW: atomic run_state.json writer (heartbeat thread)
 ├── client.py                additive verbs: --validate/--results/--monitor-json/--version-json
 └── core.py                  signal handlers → checkpoint → clean shutdown (B4)
```

The agent treats three information sources with a fixed precedence for status:
**run_state.json (truth for lifecycle)** → Redis snapshot (live detail while `running`) →
run_summary files (post-mortem detail). All three are already- or newly-specified file/CLI
contracts; none require importing V2 code.

## 4. V2 Agent API — CLI Surface

All new verbs are **additive flags** on the existing `Jarvis2` entry point (frozen surface
preserved). Fixed argv only — the agent never interpolates user text into a shell string.

| Verb | Form | Runtime cost | Purpose |
|------|------|-------------|---------|
| Version | `Jarvis2 --version-json` | none | `{package_version, api_version}` handshake |
| Validate | `Jarvis2 <task.yaml> --validate --json [--strict]` | config load only | normalized config + diagnostics; **no Redis, no workers** |
| Smoke | `Jarvis2 <task.yaml> --check-modules --json` | full small run | existing check-modules path, JSON result envelope |
| Status | `Jarvis2 --status --json (--task <yaml> \| --scan-dir <dir>)` | file read (+ optional Redis) | one-shot machine-readable status (wraps run_state + live snapshot) |
| Results | `Jarvis2 --results --json (--task <yaml> \| --scan-dir <dir>) [--columns a,b] [--limit N] [--stats]` | HDF5 read | bounded rows + summary statistics |
| Monitor | `Jarvis2 --monitor-json [--redis-host …]` | Redis read | JSON twin of `--monitor` (attach by Redis, no yaml needed) |

### 4.1 JSON envelope (all verbs)

Exactly one JSON object on stdout; human text goes to stderr only.

```json
{
  "api_version": 1,
  "kind": "validate | smoke | status | results | monitor | version",
  "ok": true,
  "data": { "...verb-specific..." },
  "error": null
}
```

- `ok=false` ⇒ `error = {"type": "<ExcClass|code>", "message": "<actionable text>"}` and
  `data` may carry partial context. Exit codes: `0` ok, `1` operation failed, `2` usage.
- `api_version` bumps only on breaking changes to the envelope, `run_state.json` schema, or
  a verb's `data` shape; additive fields do not bump it.

### 4.2 `--validate` data shape

```json
{
  "task_yaml": "/abs/path.yaml",
  "scan_name": "…", "task_result_dir": "…", "project_root": "…",
  "runtime": { "...normalized Runtime block..." },
  "sampler": {"method": "Random", "implemented": true},
  "modules": {"calculators": ["EggBox"], "operas": ["TrivialEggbox"],
               "likelihood_terms": 2, "registered_executables": ["eggboxlk"]},
  "diagnostics": [
    {"level": "error|warning", "code": "JV2-###", "path": "Runtime.redis",
     "message": "Runtime.mode is 'redis' with workers>0 but Runtime.redis is missing; …"}
  ]
}
```

Mandatory first-class diagnostics (each ties to a YAML_REFERENCE Appendix A finding):
missing `Runtime.redis` under `mode: redis` + workers>0 (**error**, A.1); unknown
`Sampling.Method` (**error**); silently-coerced values — report the rejected raw value as a
**warning** instead of hiding it (A.10); dead keys `Runtime.Subprocess`, `input[].save`
(**warning**, A.2/A.3); with `--strict`, unknown keys in known blocks (**warning**).
Validation must never bring up Redis, spawn processes, or write outside the config dict.

### 4.3 `--results` data shape

```json
{
  "db_path": "outputs/<scan>/DATABASE/samples.hdf5",
  "count": 1240,
  "columns": ["x", "y", "z", "LogL"],
  "rows": [ {"x": 0.1, "y": 0.2, "z": 0.98, "LogL": -0.02} ],
  "stats": {"LogL": {"min": -31.2, "max": -0.01},
             "best": {"LogL": -0.01, "row": {"x": 0.61, "y": 0.38}},
             "per_column": {"x": {"min": 0.0, "max": 1.0}}}
}
```

`--limit` default **50**, hard cap 1000 (the agent digests further; full extraction is a
file copy of the HDF5, not a CLI dump). Row order = best `LogL` first when `LogL` exists,
else insertion order.

## 5. `run_state.json` — Lifecycle Contract

Written by the control process to `<task_result_dir>/run_state.json`, atomically
(`tmp` + `os.replace`). Cadence: at bootstrap, then heartbeat every **5 s** (piggybacked on a
small daemon thread), then a final write at exit. Readers must tolerate unknown extra fields.

```json
{
  "api_version": 1,
  "run_id": "…", "scan_name": "…", "pid": 12345,
  "status": "preparing | running | finishing | completed | failed | interrupted",
  "counters": {"submitted": 128, "completed": 120, "failed": 2, "requeued": 1},
  "workers": {"configured": 4, "alive": 4},
  "task_yaml": "/abs/task.yaml", "task_result_dir": "/abs/outputs/scan",
  "redis": {"host": "127.0.0.1", "port": 6379, "db": 0},
  "checkpoint_file": "/abs/checkpoints/<scan>/<sampler>/state.pkl",
  "started_at": 1789… , "updated_at": 1789…,
  "last_error": null
}
```

Semantics:

- `status` transitions: `preparing → running → finishing → {completed|failed|interrupted}`.
  `interrupted` = graceful stop (§6) or checkpointed abort; `failed` = run-level exception.
- **Staleness rule (reader-side)**: `status == "running"` and `now − updated_at > 3 ×`
  heartbeat (15 s) ⇒ report `presumed_dead: true` (crash without final write). `--status`
  applies this rule itself so agent code doesn't have to.
- `redis` echoes the **effective** connection (post CLI overrides) so monitors and the agent
  can attach without re-deriving config; `null` when a fakeredis fallback was used — which
  `--validate` already flags as an error for real runs.
- The file is informational output, not an input: V2 never reads it back for decisions
  (checkpoints remain the resume source of truth).

## 6. Stop / Signal Contract

- **Graceful stop** = `SIGINT` or `SIGTERM` to the control PID (from `run_state.json`):
  the control process stops proposing, saves a runtime checkpoint (existing 30 s machinery,
  forced immediately), drains/stops the Archiver, shuts down Workers via TaskFactory, writes
  `run_state.status = "interrupted"`, exits `0`. Target: ≤ 30 s wall clock on an idle queue.
- Today the control process installs **no** signal handlers (only Workers do) — WP-D8.3
  adds them. Until then `hep_scan_stop` must be marked degraded.
- **Hard kill** (agent `mode: "kill"`): SIGKILL to the control process **group** (the agent
  starts the scan in its own process group for this purpose). Documented consequence: no
  final state write (`presumed_dead` shows it), possible staging leftovers; resume still
  works from the last periodic checkpoint. Never the default.
- Resume after any stop = the existing `Jarvis2 <yaml> --resume` (no bridge addition).

## 7. Agent-Side Tool Family (summary — spec lives in `HEP_RUNTIME_TOOLS.md`)

| Tool | Tier | Wraps | Digest (≤1K tok) |
|------|------|-------|------------------|
| `hep_validate(config, strict?)` | read¹ | `--validate --json` | error/warning counts + top diagnostics |
| `hep_smoke(config)` | exec | `--check-modules --json` | pass/fail + archived count + parity note |
| `hep_scan_start(config, resume?)` | exec (approval) | gateway background `Jarvis2 <yaml>` | run_id, pid, state-file path |
| `hep_scan_status(target)` | read | run_state.json (+ live snapshot) | status line + counters + rate |
| `hep_scan_stop(target, mode?)` | exec (approval) | SIGINT / SIGKILL-group | final status + checkpoint path |
| `hep_results(target, columns?, limit?, stats?)` | read | `--results --json` | stats + best rows; full rows → artifact |

¹ Runs a subprocess but is semantically read-only (fixed argv, no state change); it still
executes via the Gateway and is journaled. Default permission rules ship `hep_validate`,
`hep_scan_status`, `hep_results` as auto-allow; `hep_smoke`, `hep_scan_start`,
`hep_scan_stop` require the exec approval gate.

`hep_smoke` + `hep_validate` **realize and retire** the reserved `jarvis_hep_dryrun(config,
mode)` slot in Agent DESIGN.md §7.3. Per-sample debugging needs no new tool: `hep_results` /
`hep_scan_status` return concrete paths (`SAMPLE/<uuid>/`, `logs/samples/<uuid>.log`) that
the existing `fs_read` consumes.

## 8. Version & Capability Handshake

First `hep_*` call in a session runs `Jarvis2 --version-json`; the tool caches
`{package_version, api_version}` for the session. `api_version` mismatch ⇒ every `hep_*`
tool returns `ok=false` with an actionable "upgrade Jarvis-HEP-v2 / Jarvis-Agent" message —
never a silent fallback to text parsing.

## 9. Work Breakdown

### V2 side — milestone **D8 "Agent Bridge"** (details in `V2_DISTRIBUTED_PLAN.md` §WP-D8.x)

| WP | Delivers | Blocks |
|----|----------|--------|
| D8.1 | `jarvishep2/api.py` (envelope + validate/results impl) + `--version-json` | — |
| D8.2 | `run_state.py` writer wired into `Jarvis2Core` lifecycle + `--status`/`--monitor-json` | D8.1 |
| D8.3 | control-process signal handlers → checkpoint-on-SIGINT/SIGTERM (stop contract §6) | — |
| D8.4 | strict-validate diagnostics (silent-coercion warnings, dead keys, unknown keys) | D8.1 |

### Agent side — milestone **M4.5 "Scan Runway"** (details in `HEP_RUNTIME_TOOLS.md`)

`hep_*` tool family + permission defaults + functional-checklist items; depends on **V2 D8.1–
D8.3 shipped** (cross-repo dependency; D8.4 upgrades diagnostics but does not block).
Verification gate: from a validated `proposals/*.yaml`, drive validate → smoke → start →
poll status → results digest → graceful stop end-to-end through the TUI with the real local
model, all runs journaled.

## 10. Non-Goals & Open Questions

**Non-goals (this iteration):** no long-lived V2 daemon or REST/MCP server (MCP re-entry
condition tracked in Agent DESIGN §10.2); no remote/multi-host scan control (Redis already
allows remote monitor attach — control stays local); no write-path into a *running* scan
(parameter steering is future work); no V1 support (`Jarvis` 1.7.4 is frozen).

**Open questions:**

1. Should `--results` grow server-side filtering (`--where "LogL > -5"`)? Deferred — the
   agent can post-filter ≤1000 rows; revisit when scans exceed ~10⁶ rows.
2. Multi-scan registry (agent juggling several concurrent scans) — `run_state.json` per scan
   dir suffices for now; a `~/.jarvis` scan index is an agent-side concern if needed.
3. Event push (file-watch or Redis pub/sub) instead of polling — polling `run_state.json`
   at ≥5 s granularity is fine for slow-regime scans; revisit for fast regimes.

## 11. Cross-References To Update When Implementing

- `Jarvis-Agent/docs/DESIGN.md` §7.3 tool table (hep_* family row block), §10.1 milestone
  table (M4.5), Appendix C doc map — **done in this change**.
- `Jarvis-Agent/docs/HEP_RUNTIME_TOOLS.md` — agent-side spec (**created in this change**).
- `V2_DISTRIBUTED_PLAN.md` milestone map + ledger + WP-D8.1…D8.4 — **done in this change**.
- `YAML_REFERENCE_2.0.md` — when D8.4 lands, move the "silent coercion" review findings from
  Appendix A into documented `--validate` diagnostics.
- Agent Schema KB (`kb/jarvis-hep/`): regenerate for the V2 runtime from
  `YAML_REFERENCE_2.0.md` with a `jarvis2@<commit>` version stamp (Agent DESIGN Appendix B
  currently describes the V1 schema).
