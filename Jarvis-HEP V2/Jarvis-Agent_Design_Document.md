# Jarvis-Agent — Design Document

**Version**: v0.3 (2026-07-03)  
**Status**: Initial Design + Model Evaluation; V2 integration contract now fixed by
[`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md)  
**Platform**: macOS (Apple Silicon) only  
**LLM Backend**: MLX-LM  
**Relationship with Jarvis V2**: Upper orchestration layer (Option A)

---

## 1. Overview

**Jarvis-Agent** is a local intelligent agent system designed to assist researchers in using **Jarvis-HEP V2**. It runs entirely on macOS using MLX-LM for local inference and provides high-level capabilities such as:

- Understanding research goals (e.g., NMSSM scans, μ problem, parameter optimization)
- Generating and modifying Jarvis task YAML configurations
- Submitting, monitoring, and managing scans via Jarvis V2
- Analyzing results and suggesting next steps
- Maintaining memory of past experiments and user preferences

The agent acts as an **intelligent orchestration layer** on top of Jarvis V2, without modifying V2’s core execution engine.

---

## 2. Strategic Decision: Separation from Jarvis V2

**Decision (2026-06-27)**: We adopt **Option A**.

- **Jarvis V2** remains a pure, reliable distributed execution engine (Redis + long-lived Workers + Archiver).
- **Jarvis-Agent** is developed as a **separate project/layer** that calls Jarvis V2 through well-defined interfaces (primarily Redis + filesystem).
- This keeps V2 focused on performance, reliability, and correctness, while allowing Jarvis-Agent to evolve more aggressively in planning, tool use, and user interaction.

---

## 3. Goals & Non-Goals

### Goals (MVP)

- Run fully locally on macOS using MLX-LM
- Reliably call Jarvis V2 (submit tasks, monitor status, read results)
- Generate valid and reasonable Jarvis YAML configurations
- Support basic agentic workflows (Plan → Execute → Observe → Reflect)
- Provide good structured output and tool-calling reliability
- Maintain useful memory of experiments

### Non-Goals (MVP)

- No Windows/Linux support
- No multi-agent collaboration (single agent first)
- No heavy long-term vector database (start simple)
- No deep autonomous scientific discovery without human oversight
- Avoid modifying Jarvis V2 core

---

## 4. Architecture Overview

```
User (Natural Language)
        ↓
JarvisAgent (Main Loop)
        ├── LLM Planner (MLX-LM)
        ├── Tool Registry
        │   ├── JarvisTool (submit, monitor, read results)
        │   ├── ConfigGenerator
        │   ├── ResultAnalyzer
        │   ├── ParameterSuggester
        │   └── MemoryTool
        ├── Memory Store
        └── Jarvis V2 Client (Redis + Filesystem)
                ↓
        Jarvis V2 (Execution Engine)
```

**Key Principles**:
- The Agent interacts with Jarvis V2 **only through explicit Tools**.
- LLM is responsible for planning and decision-making; execution is handled by reliable Tools.
- All actions should be auditable and reproducible.

---

## 5. Integration with Jarvis V2

**Recommended Interface** (in order of preference):

| Method              | Description                                      | Recommendation | Notes |
|---------------------|--------------------------------------------------|----------------|-------|
| **Redis + Filesystem** | Push tasks via Redis, monitor via `op_count` + heartbeat, read results from `DATABASE/` and `SAMPLE/` | **Strongly Recommended** | Most powerful and real-time |
| CLI (`subprocess`)  | Call `Jarvis2` commands                          | Acceptable     | Simpler but less elegant |
| Direct Python Import| Import V2 modules directly                       | Not Recommended| Breaks process boundaries |

**Primary Integration Path**: Redis + Filesystem hybrid.

> **Update (2026-07-03).** The concrete contract is now fixed by
> [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md): machine-readable `Jarvis2`
> CLI verbs (`--validate/--results/--status --json`), a `run_state.json` lifecycle file, a
> SIGINT graceful-stop contract, and read-only Redis for live monitoring. "Direct Python
> Import" stays rejected. V2-side work = plan milestone D8; agent-side tools =
> `Jarvis-Agent/docs/HEP_RUNTIME_TOOLS.md` (milestone M4.5).

---

## 6. LLM Backend & Model Selection

### Current Evaluation (as of 2026-06-27)

We are evaluating **MoE models** quantized for MLX due to their favorable memory/speed vs capacity trade-off on Apple Silicon.

**Models under consideration**:

| Model                                              | Type          | Active Params | Tested Speed | Strengths                          | Status |
|----------------------------------------------------|---------------|---------------|--------------|------------------------------------|--------|
| `unsloth/Qwen3.6-35B-A3B-UD-MLX-4bit`              | General (UD)  | ~3B           | **20 t/s**   | Good general reasoning             | Tested |
| `mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit`  | Coder/Instruct| ~3B           | TBD          | Stronger tool calling & structured output | Under evaluation |

### Current Recommendation

- **Primary candidate**: `mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit`
  - Better suited for tool calling, YAML generation, and structured output — critical for Jarvis-Agent.
- **Backup**: `unsloth/Qwen3.6-35B-A3B-UD-MLX-4bit` (already validated at 20 tokens/s).

**Rationale**: For an Agent system, reliable tool use and structured generation are more important than raw general knowledge in the early stages.

---

## 7. Core Components (Proposed)

```
jarvis_agent/
├── agent.py                 # Main agent loop + orchestration
├── llm/
│   └── mlx_wrapper.py       # MLX-LM abstraction layer
├── tools/
│   ├── base.py
│   ├── jarvis_tool.py       # Core bridge to Jarvis V2
│   ├── config_tool.py       # YAML generation & validation
│   ├── analysis_tool.py     # Result analysis
│   └── memory_tool.py
├── memory/
│   └── store.py             # Short-term + long-term memory
├── prompts/
│   └── system_prompt.py
├── config.py
└── cli.py
```

---

## 8. Tool / Action Space (Initial Set)

| Tool Name                | Purpose                              | Input                          | Output                     | Priority |
|--------------------------|--------------------------------------|--------------------------------|----------------------------|----------|
| `submit_scan`            | Submit a Jarvis task                 | YAML content or path           | task_id / uuid             | High     |
| `get_scan_status`        | Get current scan status              | task_id                        | status + statistics        | High     |
| `analyze_results`        | Analyze completed results            | task_id or path                | Structured analysis        | High     |
| `suggest_next_parameters`| Suggest next set of parameters       | History + goal                 | Parameter suggestions      | High     |
| `generate_config`        | Generate Jarvis YAML from goal       | Natural language description   | Complete task YAML         | High     |
| `read_sample_log`        | Read detailed log of a Sample        | uuid                           | Log content                | Medium   |
| `memory_write` / `read`  | Read/write agent memory              | key + content                  | -                          | Medium   |

> **Update (2026-07-03).** The V2-facing rows above (`submit_scan`, `get_scan_status`,
> `analyze_results`, `read_sample_log`) are realized as the native `hep_*` tool family —
> `hep_validate / hep_smoke / hep_scan_start / hep_scan_status / hep_scan_stop /
> hep_results` — specified in `Jarvis-Agent/docs/HEP_RUNTIME_TOOLS.md`, on top of the wire
> contract in [`DESIGN_AGENT_BRIDGE_2.0.md`](DESIGN_AGENT_BRIDGE_2.0.md).

---

## 9. Memory Strategy (MVP)

- **Short-term memory**: Current conversation + active task context (injected into prompt)
- **Long-term memory** (MVP): Simple structured storage (JSONL or SQLite)
  - Store: user goals, past scans, key conclusions, failure reasons, user preferences
- Future upgrade path: Local vector database (Chroma / LanceDB) if needed

---

## 10. Development Roadmap

| Phase | Focus                                      | Key Deliverables                          | Status |
|-------|--------------------------------------------|-------------------------------------------|--------|
| 0     | Design + Model Evaluation                  | This document + model testing             | Current |
| 1     | Core Framework + LLM Wrapper               | Basic agent loop + MLX integration        | Planned |
| 2     | JarvisTool + Config Generation             | submit_scan, generate_config tools        | Planned |
| 3     | Monitoring + Result Analysis               | get_status, analyze_results tools         | Planned |
| 4     | Memory + Multi-turn Iteration              | Basic memory system + reflection loop     | Planned |
| 5     | Advanced Planning & Human-in-the-loop      | Stronger reflection + user confirmation   | Future |

---

## 11. Open Questions & Decisions

1. **Model Finalization**
   - Should we lock `Qwen3-Coder-30B-A3B` as the primary model, or keep both models switchable?

2. **Tool Calling Format**
   - Use native function calling (if supported well by the model) or structured JSON prompting?

3. **Human-in-the-loop Strategy**
   - How much autonomy should the agent have by default? (Especially for submitting scans)

4. **Memory Scope**
   - Should memory be per-project or global across all user interactions?

5. **Evaluation Criteria**
   - What are the key success metrics for the Agent in the first usable version? (e.g., successful YAML generation rate, tool calling accuracy, user effort reduction)

---

## 12. Summary of Key Decisions (2026-06-27)

- Jarvis V2 stays as pure execution engine (Option A).
- Jarvis-Agent is a separate local macOS + MLX project.
- Prefer MoE models (~3B active parameters) for memory efficiency.
- `Qwen3-Coder-30B-A3B-Instruct` is currently the leading candidate due to stronger tool use and structured output capabilities.
- Integration with V2 will primarily go through Redis + filesystem.

---

**Document Status**: Living document. Will be updated as model evaluation and architecture decisions progress.

---

*Generated based on discussion on 2026-06-27*