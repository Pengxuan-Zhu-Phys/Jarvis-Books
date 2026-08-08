# DESIGN — Multi-mode Calculators (`Calculators.Modules[].modes`, V2, D20)

**Status**: implemented, shared-only; D20.5–D20.8 hardening completed in the working tree
**Scope**: V2-only; do not backport to V1
**Last updated**: 2026-08-02
**Review evidence**: [`D20_MULTIMODE_REVIEW_2026-08-02.md`](D20_MULTIMODE_REVIEW_2026-08-02.md)

---

## 1. Decision

A multi-mode calculator represents one physical software installation whose
uses are mutually exclusive. Switching use requires changing and rebuilding
that same installation. Prospino processes are the reference case.

V2 implements one model only:

> **One parent calculator owns one physical PackID pool. Each mode is a logical
> execution step. A held PackID is rebuilt into the requested mode when needed.**

There is no `per_mode` option, `mode_packs` switch, `@Mode` token,
`${mode_dir}` token, or mode-specific physical PackID directory. These appeared
in an earlier design draft and are intentionally not part of the interface.
`@Mode` and `${mode_dir}` are rejected by `JV2-MOD-006`.

Multimode must not be used merely because two commands share source code. If
both uses can coexist in one installation—different micrOMEGAs subdirectories,
different MadGraph process directories, or two independent binaries—declare
two normal calculator modules. Multimode exists only for mutually exclusive
build states: using B necessarily replaces A in the same runtime directory.

## 2. Why shared-only

For a package with M modes and P PackID slots:

| Model | Physical directories | Runtime cost |
| --- | ---: | --- |
| per-mode | M × P | each directory builds once |
| shared-only | P | rebuild only when a PackID changes mode |

Prospino may expose many processes while rebuilding quickly. Creating
`modes × packs` complete copies defeats the purpose of multimode. A shared pool
keeps disk use proportional to PackID count, while affinity scheduling reduces
unnecessary rebuilds.

The shared model is safe because Redis grants exclusive ownership of a physical
PackID. Modes of the same parent are serialized within one sample; different
calculator parents remain eligible for concurrency.

## 3. Canonical YAML

```yaml
Calculators:
  Pools:
    Prospino: 8

  Modules:
    - name: Prospino
      path: "../runtime/Prospino/@PackID"
      source: "../source/Prospino"
      clone_shadow: true

      # Shared base. It runs only for a new/invalidated PackID base.
      installation:
        - "cp -R ${source}/. ${path}"

      # Optional; runs before every selected mode call.
      initialization:
        - "rm -f ${path}/prospino.dat"

      modes:
        - name: ng

          # Optional mode rebuild. It does not replay parent.installation.
          installation:
            - "python configure.py ng"
            - "make"

          # Optional; parent initialization runs first, then this block.
          initialization:
            - "rm -f ng.out"

          # Optional execution-directory fallback.
          path: "${path}/run-ng"

          execution:
            # Highest-priority execution directory.
            path: "${path}/run-ng/current"
            commands:
              - "./prospino"
            output:
              - name: xsec_ng
                path: "@Sdir/ng.json"
                type: JSON
                variables: [{name: xsec_ng}]

        - name: ns
          installation:
            - "python configure.py ns"
            - "make"
          execution:
            commands: ["./prospino"]
            output:
              - name: xsec_ns
                path: "@Sdir/ns.json"
                type: JSON
                variables: [{name: xsec_ns}]
```

The parent `path` is always the shared physical installation anchor and must
contain `@PackID`. A mode may declare `path` and `execution.path`, but those
fields select only the command working directory. Execution-directory
precedence is:

```
mode.execution.path > mode.path > parent.path
```

No form creates `Prospino/<mode>/<PackID>`; the physical directory remains, for
example, `../runtime/Prospino/001`.

## 4. Declaration expansion

`modes` is user-facing declaration syntax. After schema and semantic validation,
the loader expands it into dotted logical module names:

```text
Prospino + modes [ng, ns]
    -> Prospino.ng
    -> Prospino.ns
```

Each expanded child carries internal metadata identifying the shared parent and
target mode. Both children retain the same parent physical path and pool name.
Normal calculators without `modes` are unchanged.

### 4.1 Inheritance and lifecycle blocks

- The parent owns physical fields: `source`, `deps_source`, `clone_shadow`,
  `make_paraller`, and `symlink_name`. A mode cannot override them.
- A mode may override logical/runtime fields such as `env_setup`, `timeout`,
  `selection`, `required_modules`, and execution path.
- Parent `execution` is forbidden. Every mode owns its `execution` block.
- Parent and mode `installation` are separate stages, not one replayed list.
- Parent initialization and mode initialization are concatenated for every
  selected call, parent first.

### 4.2 Dependencies

The dot is the module/mode separator:

```yaml
required_modules: [Prospino.ng]       # one explicit mode
required_modules: [Prospino.ng, Prospino.ns]
required_modules: [Prospino]          # all modes of this parent
```

A bare multimode parent expands to all children except the child itself. Thus
`Prospino.ns` with `required_modules: [Prospino]` depends on sibling modes but
does not acquire a self-dependency.

Bare dependency names are not owned exclusively by Calculators; production
cards legitimately reference LibDeps modules, Operas modules, and the reserved
`Parameters` pseudo-module. `JV2-MOD-005` therefore validates only dotted names
whose parent is a declared multimode calculator. `Prospino.typo` is rejected
with a did-you-mean suggestion; bare `LoopTools`, `BuildBSMPTInput`, and
`Parameters` are allowed.

## 5. Installation lifecycle and disk truth

For a requested mode, Worker preparation follows this lifecycle:

1. Acquire exclusive ownership of a physical parent PackID from Redis.
2. Read the PackID's `.jarvis_install_stamp.json`.
3. Run parent `installation` only when the physical base is new, explicitly
   reinstalled, or its parent fingerprint/epoch changed.
4. If the trusted stamp mode differs from the requested mode, run only that
   mode's optional `installation`.
5. Write the successful parent fingerprint, epoch, and mode into the stamp.
6. Run parent initialization and then mode initialization.
7. Run mode execution in the resolved execution directory.

Before any rebuild, the old stamp is deleted. A crash can therefore leave an
unassigned/invalid pack, never a stamp that claims mode A while the directory is
a half-built mode B. The stamp is written only after successful preparation.

All modes of one parent share one `jarvis_install.json` and one reinstall epoch.
Setting `reinstall: true` invalidates the shared base once; it does not advance
the epoch once per mode. On process restart, pool registration reads successful
disk stamps and restores `pack -> mode` affinity. Redis is the live concurrency
authority; the stamp is persistent truth and recovery fallback.

## 6. Redis model

For parent `Prospino`, Redis uses:

```text
calc:free:Prospino:mode:ng
calc:free:Prospino:mode:ns
calc:free:Prospino:unassigned
calc:busy:Prospino
calc:packmode:Prospino
```

A free PackID belongs to exactly one mode-affinity list or the unassigned list.
`calc:busy:Prospino` guarantees exclusive physical ownership. Successful
release atomically records the trusted target mode and returns the pack to that
mode's free list. Preparation failure returns it to `unassigned`.

### 6.1 Acquire preference and bounded warm wait

Acquisition order is:

1. a free pack already warm for the target mode;
2. an unassigned/never-built pack;
3. if a target-mode pack is currently busy, wait for it for a bounded interval;
4. only then borrow the most plentiful other-mode pack and rebuild it.

The default warm wait is three seconds and never exceeds the normal calculator
acquire timeout. Waiting occurs only when a target-mode pack is actually in
flight. If no target pack exists, the Worker borrows immediately; this avoids
penalizing pools smaller than the mode count.

This wait is essential under contention. Without it, a Worker immediately
destroys another useful affinity while its exact warm pack is about to return;
multi-key BLPOP cannot preserve preference after blocking because whichever key
is pushed first wins.

### 6.2 Worker ordering

For modes of the same parent in one dependency layer, a Worker executes them
serially and greedily:

1. Prefer the pending mode with the most free warm packs.
2. If no warm pack is free, prefer the mode with the fewest packs currently
   busy for that target, spreading cold starts across Workers.
3. Acquire using §6.1 and execute the selected mode.

This naturally staggers Workers: one sample may run `ng` first while another
runs `ns` first. There is no static ratio. Redis state and current contention
determine the order at every step.

Different calculator parents in the same layer may still run concurrently.
Dependency layers remain authoritative; the greedy rule never reorders steps
across layers.

### 6.3 Pool sizing

Affinity cannot remove a physical capacity limit. Observed guidance is:

```text
Calculators.Pools.<parent> >= mode count + Worker count
```

This is a performance recommendation, not a correctness requirement. When the
pool is approximately that size, steady-state contention can reach zero mode
rebuilds. A pool equal only to mode count can still degenerate under multiple
Workers. If PackID count is smaller than mode count, rebuild thrashing is
structurally unavoidable because every mode cannot remain resident.

`JV2-MOD-009` reports an undersized pool as a warning and suggests a concrete
size. It does not reject the task. Users may deliberately accept rebuilds to
save disk.

## 7. Selection semantics

Mode selection is evaluated before Redis acquisition and before greedy mode
ordering. A mode whose `selection` is false:

- does not acquire or hold a PackID;
- does not run parent or mode installation;
- does not run initialization or execution;
- does not rewrite the pack's affinity label;
- does not record a misleading mode PackID for the sample.

This ordering is required because parameter-dependent mode selection is a
primary multimode use case. Preparing first would rebuild for a mode that never
ran and would pollute affinity for peer Workers.

## 8. Flowchart semantics

The workflow graph contains separate logical nodes such as `Prospino.ng` and
`Prospino.ns`, because each mode owns distinct I/O and dependency edges. These
nodes do not represent separate physical packs.

The scene JSON records a `shared_runtime` group and per-node metadata containing
the parent pool and `serialized_with_sibling_modes` constraint. The current
visual renderer does not draw an external dashed container; the grouping stays
semantic so the graph remains uncluttered. Mode labels state the shared PackID
and serialization relationship.

## 9. Validation contract

| Code | Rule |
| --- | --- |
| `JV2-MOD-001` | Parent/mode names cannot use `.` internally; dot is the separator. |
| `JV2-MOD-002` | A multimode parent cannot define `execution`. |
| `JV2-MOD-003` | Parent and sibling mode names must be unique. |
| `JV2-MOD-004` | Sibling modes cannot publish duplicate output names. |
| `JV2-MOD-005` | A dotted mode under a known multimode parent must exist; includes did-you-mean. |
| `JV2-MOD-006` | Shared physical contract: `clone_shadow`, `@PackID`, physical overrides, and forbidden `@Mode`/`${mode_dir}`. |
| `JV2-MOD-007` | `Calculators.Pools` must use the physical parent, not `Parent.mode`. |
| `JV2-MOD-008` | Pool parent must name a declared calculator. |
| `JV2-MOD-009` | Warning for pool capacity likely to cause affinity thrashing. |

The schema surface is closed. Unknown keys and misspellings fail before Redis,
Workers, output directories, or installation side effects start.

## 10. Failure and recovery rules

- Rebuild start deletes the old stamp.
- Successful preparation sets `runtime_ready`; a later execution failure may
  return the pack to the newly prepared target-mode list.
- Preparation failure returns the pack to `unassigned`.
- Redis release is atomic; socket failure retains local ownership for watchdog
  recovery instead of silently double-freeing the pack.
- Worker/process cleanup force-releases held shared packs to `unassigned` when
  the trusted mode cannot be established.
- Restart registration restores only modes found in successful stamps.

## 11. Compatibility and non-goals

- V1 is unchanged. V1's unfinished multimode stub is not a compatibility target.
- Ordinary single-mode calculators keep their original Redis keys, pools,
  installation lifecycle, workflow behavior, and YAML.
- Multimode is not extended to Operas; Operas can declare multiple normal
  functional modules without a mutable native build.
- There is no global per-mode `build` stage. Expensive globally reusable builds
  belong in `LibDeps`; mode `installation` is the PackID-local switch step.
- There is no user-configurable `per_mode/shared` strategy in V2.

## 12. Work packages and acceptance

| WP | Result | Acceptance |
| --- | --- | --- |
| D20.1–D20.4 | shared-only multimode implementation | expansion, schema, Redis, Worker, flowchart, real-process E2E |
| D20.5 | dependency-validation regression fix | 65 shipped cards have zero `JV2-MOD-*` errors; bad known dotted mode still rejected |
| D20.6 | contention hardening | bounded target-mode wait; 3 modes/3 packs/3 Workers preserve affinity; sizing warning/documentation |
| D20.7 | selection before acquire | skipped mode never touches Redis, build, or affinity |
| D20.8 | polish and design synchronization | Utils migration wording restored; self-dependency removed; this document matches code |

Rollback remains simple: cards without `modes` never enter the shared-mode path.
