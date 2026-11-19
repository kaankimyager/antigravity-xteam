# Model Roles, Plan-Agnostic Bucketing & Failover Matrix

## 1. Plan-Agnostic Discovery (`swarm`)

Always call `swarm(action="models", all=true)` before `swarm(action="add")`. Classify live models dynamically using `quota_resets` and `quotas` so the workflow adapts to any plan or future model release:
1. **Short-Window Heavy Pool** (models resetting in hours, e.g., `gemini-4-neon`, `gemini-pro-orchestrator-neon`, `gemini-4-hydrogen`, `gemini-3.8-flash`): Use for `Scout`, `Builder`, `Architect`, and `Standard Reviewer`.
2. **Independent Weekly Pool** (models with their own multi-day reset, e.g., `gemini-4-argon`): Use for `Architect` escalations and secondary reviews without touching combined pools.
3. **Combined Scarce Pool** (e.g., `claude-sonnet`, `claude-opus`, `gpt-oss`): Reserve strictly for `xteam audit` or high-risk complex diffs (max 1 batched call/task).

| Role | Primary Match Pattern | Mode | Secondary / High-Risk Match |
| :--- | :--- | :--- | :--- |
| **Scout** | `*flash*low` or `*flash*medium` | `low` / `medium` | `*hydrogen*low` or `*neon*low` |
| **Builder** | `*neon*medium` or `*hydrogen*medium` | `medium` | `*flash*medium` or `*argon*medium` |
| **Architect** | `*orchestrator-neon*` or `*neon*medium` | `medium` | `*argon*medium`, `*orchestrator-g1-pro*`, or `*opus*medium` (last resort) |
| **Reviewer (Standard)** | Cross-check `*hydrogen*medium` <-> `*neon*medium` | `medium` | `*argon*medium` (static tools first: `tsc`/`lint`/`test`) |
| **Reviewer (High-Risk)** | `*sonnet*low` or `*sonnet*medium` | `low` / `medium` | `*gpt-oss*medium` (max 1 batched call/task) |

- Reject `*high` and `*xhigh` variants unless explicitly requested by the user.
- **Swarm Boot Cost:** Spawning a `Claude` or `GPT` worker loads full tool schemas and board context. Never spawn them for trivial/single-file checks or multi-turn back-and-forth; batch all diffs into 1 prompt.

## 2. Dynamic Cross-Family Failover Matrix

Apply `< 30%` soft-shift (offload routine work to `Neon`/`Hydrogen`/`Argon`) and `<= 5%` hard failover:

| Exhausted Pool (`<= 5%`) | Takeover Role & Model | Execution Rule |
| :--- | :--- | :--- |
| **Claude & GPT Pool `<= 5%`** | `Architect` & `High-Risk Reviewer` -> `gemini-4-neon-medium` / `gemini-4-argon-medium` (fallback `gemini-4-hydrogen-medium`) | Do not invoke `Claude` or `GPT`; use `Neon` (5h) or `Argon` (separate weekly pool). |
| **Gemini Argon `<= 5%`** | `Architect` -> `gemini-pro-orchestrator-neon` / `gemini-4-neon-medium`; `Critical Reviewer` -> `claude-sonnet-5-5-low` (if `> 15%`) or `Hydrogen` | Route architecture to `Neon` / `Hydrogen` without touching weekly quotas. |
| **Gemini Neon / Hydrogen `<= 5%`** | `Builder` -> the remaining healthy 5h heavy model (`Hydrogen` <-> `Neon`), then `gemini-3.8-flash-medium` (or `gemini-4-argon-medium` for complex modules) | Rotate between `Neon` and `Hydrogen` first; if both are low, split edits for `Flash`. |
| **Gemini Flash `<= 5%`** | `Scout` -> `gemini-4-hydrogen-low` or `gemini-4-neon-low` | Enforce strict `rg -n` + narrow line-window reads. |
| **All Pools `<= 5%` (or `Neon` + `Hydrogen` + `Flash` `<= 5%`)** | Route to whichever single pool remains `> 5%` (`Argon` or `Claude Low`); if **none** remain `> 5%`, enter **Single-Agent Ultra-Lean Mode** | Stop spawning `swarm` workers, use `rg -n` + minimal line reads only, and warn the user of quota exhaustion. |

## 3. Hot-Swap Before Compact (`swarm(action="model")`)

When an active worker hits a quota or capacity limit mid-task:
- **Short/Medium Session (`<= 15–20` messages):** Use `swarm(action="model", name="...", model="<fallback>")` to preserve conversation state.
- **Bloated Session (`> 15–20` messages):** Do **not** hot-swap (carrying large history will burn the fallback's quota). Ask for a 5-line handoff summary, `swarm(action="kill", name="...")`, and spawn the fallback worker with a clean context.

## 4. `swarm` vs. `invoke_subagent` & Parallel Rules

- Use `swarm` for explicit model pinning; use `invoke_subagent` for `browser` (`Model: "flash"`), `image-generator`, and read-only `research`.
- **Parallel Builders (`workspace="share"`):** Author shared contracts/types first, assign disjoint paths, and freeze install/build commands until all parallel edits merge.

## 5. Concise Worker Brief Templates

### Scout Brief
```text
Role: Scout. Inspect <target_paths> via rg -n and narrow line-range reads.
Return: 1) File paths + line numbers, 2) Concise symbol summary, 3) Blockers. Do not edit files.
```

### Builder Brief (Single or Parallel)
```text
Role: Builder. Own ONLY <assigned_paths>. Do not modify files outside your scope.
Write code only within your assigned paths; do NOT run package install or build commands (e.g., npm install, npm run build) during parallel execution—a single designated Builder or Coordinator runs install/build after merging.
Goal: <concise_spec>. Done when: <acceptance_criteria>.
```

### Reviewer Brief (Diff-Only, Batched)
```text
Role: Reviewer. Inspect ONLY the provided git diff / changed symbols below (do not scan unrelated files).
Static checks (<build/test_cmd>) already passed. Check for logic bugs, edge cases, and security issues.
Return: concise bullet list of defects with file:line, or "LGTM".
```

## 6. Orphan Worker & Daemon Cleanup

- **Stalled Workers:** Check `swarm(action="status")`, diagnose errors, `swarm(action="kill", name="...")`, and allow at most 1 retry.
- **Browser & Daemon Cleanup:** Verify build passes -> start server (`IsDaemon=true`) -> pass `http://localhost:<port>` + steps to `browser` -> immediately `manage_task(action="kill", TaskId="<id>")` after `browser` finishes.
