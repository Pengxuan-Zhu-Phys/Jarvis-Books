# DESIGN — Cluster Execution & Broker Durability (V2, D14)

**Status**: design accepted 2026-07-16; no code yet — all WPs `todo`
**Date**: 2026-07-16
**Scope**: run Workers on hosts other than the control process — standalone worker
pools first, batch-system submitters second — plus the Redis durability/auth work that
multi-host operation makes mandatory (DESIGN §13.5 open question).
**Maintainer constraint**: **D8 stays parked** — remote control uses the human CLI only.

---

## 1. Problem

HEP scans run on SLURM/HTCondor clusters; V2 today is single-host: `EnvReqs.V2.redis`
can point at a remote broker (D12.4), but `TaskFactory` only spawns local Workers, the
worker spawn template travels via spawn-pickle, and the managed `Jarvis-Redis:<scan>`
instance is unauthenticated, non-persistent, and bound locally.

## 2. Goals

1. **Phase 1 — manual pools**: `Jarvis2 worker start --connect host:port[/db]`
   launches a self-supervised Worker pool on any node that shares the filesystem.
2. **Config over the broker, not pickles**: the control process publishes the complete
   worker spawn template (JSON, no live objects — invariant 7/8 already guarantee
   picklable-light configs) under `hep:worker:template:<scan>`; remote pools pull it.
   One source of truth, no YAML copying to nodes.
3. **Remote-aware supervision**: the factory watchdog **sweeps** dead foreign workers
   (slots, orphan PGIDs on their host are handled by the pool's own local watchdog) but
   **never respawns** them; each pool supervises its own children. Worker ids become
   `<host>:<n>` to keep `hep:worker:status:{id}` collision-free.
4. **Broker hardening**: optional `requirepass` auth (`EnvReqs.V2.redis.password` or
   env var `JARVIS_REDIS_PASSWORD` — never logged), AOF persistence knob for the
   managed instance, and documented resume semantics across a broker restart.
5. **Shared-FS invariant, stated not assumed**: `@Sdir`, staging, calculator runtime
   dirs, and the DATABASE all require a shared filesystem across nodes. Non-shared FS
   (object-store staging) is explicitly out of scope for D14.
6. **Phase 2 — submitters**: `Jarvis2 cluster submit --backend slurm` renders sbatch
   scripts that call the Phase-1 command; HTCondor analogous. Submitters generate and
   submit; they do not babysit jobs (squeue is the user's tool; `Jarvis2 monitor`
   remains the scan-level view).

### Non-goals

- Kubernetes / cloud object storage / non-shared-FS operation.
- Cross-site (multi-cluster) scans.
- Elastic autoscaling policy (manual pool sizing only; pools may join/leave mid-run).
- Agent-driven cluster control (parked with D8).

## 3. Design notes

- **Join protocol**: a pool validates `scan_name` + config hash from the template key
  before starting Workers, so a stale node cannot join the wrong scan. Template is
  written before sampling starts and deleted at teardown.
- **Heartbeats already work remotely** — they are Redis-only. `stale_sec` gains a
  cluster-safe default note (network jitter) in YAML_REFERENCE.
- **PackID pools are global by design**: `calc:free:<name>` capacity spans all hosts;
  per-host calculator installs rely on the shared FS + idempotent installation
  convention (YAML_REFERENCE §9.4).
- **Managed vs external Redis**: with `redis.host` ≠ localhost the control process
  never starts `Jarvis-Redis:<scan>`; with AOF enabled the managed instance writes
  under `<scan>/redis/` so `--resume` after a broker crash replays bucket/slot state
  consistently with the runtime checkpoint (checkpoint stays authoritative; Redis
  replay is an optimization, and a wiped broker must still resume from checkpoint).

## 4. Work packages

| WP | Title | Depends on | Accept |
|---|---|---|---|
| D14.1 | Worker template over Redis + `Jarvis2 worker start --connect` | — | two pools on one host (simulating two nodes) drain a live Eggbox scan; DATABASE parity with single-host golden; template deleted at teardown |
| D14.2 | Remote-aware watchdog + `<host>:<n>` worker ids | D14.1 | killing a foreign pool mid-run: slots swept, samples re-queued, control run completes; no respawn attempts logged |
| D14.3 | Broker auth + AOF knob + broker-restart resume semantics | — | scan with `requirepass` end-to-end; kill managed Redis mid-run → `--resume` completes from checkpoint; password absent from every log/`ps` title |
| D14.4 | `Jarvis2 cluster submit --backend slurm` (+ HTCondor) | D14.1 | rendered sbatch runs Phase-1 pool untouched on a real cluster; dry-run mode prints the script; docs in INSTALL.md |

**Rollback**: no `--connect` / no `cluster` block → exactly today's single-host
behavior. **Out of scope**: non-shared FS, autoscaling, K8s, cross-site.

## 5. Risks

1. **Version skew** between control and pool hosts — template carries
   `jarvishep2.__version__`; pools refuse a major/minor mismatch.
2. **Slot sweep vs remote orphans**: the factory cannot `killpg` on another host —
   that duty moves to the pool-local watchdog (same D12.7 code, running in the pool
   supervisor); document the gap for hard-killed pools (stale slots recovered by
   heartbeat sweep, directories cleaned by idempotent installation).
3. **NFS latency** on staging/DATABASE — Archiver batching already amortizes; the
   acceptance run must include an NFS-mounted output dir before D14 closes.
