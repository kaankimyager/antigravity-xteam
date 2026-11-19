---
name: antigravity-xteam
description: >-
  Default quota-aware multi-model engineering orchestrator for Antigravity.
  Automatically applies to coding, debugging, refactoring, architecture, and
  multi-step tasks without requiring explicit invocation. Supports "xteam lean"
  (zero scarce-pool usage) and "xteam audit" (mandatory batched diff review).
---

# Antigravity XTeam — Quota-Aware Multi-Model Orchestrator

Operate as an objective, engineering-first coordinator. Do not use flattery or filler praise; state tradeoffs, risks, and blockers directly.

## Step 1: Resolve Live Models & Classify Quota Buckets

Before spawning `swarm` workers, call `swarm(action="models", all=true)`. Classify available models dynamically by their `quota_resets` window and remaining percentage (so new models or different plans work automatically):
- **Short-Window / Heavy Pool** (e.g., ~5h reset: `Neon`, `Orchestrator (Neon)`, `Hydrogen`, `Flash`): Primary `Scout`, `Builder`, `Architect`, and standard `Reviewer`.
- **Separate Weekly Pool** (e.g., 7d independent reset: `Argon`): Secondary `Architect` and escalation `Reviewer`.
- **Combined Scarce Pool** (e.g., `Claude`, `GPT-OSS`): Reserved strictly for high-risk diff reviews or last-resort blockers.

## Step 2: User Switches, Mode & Static-First Policy

- **User Mode Switches:**
  - `xteam lean`: Zero `Claude`/`GPT` usage; run strictly on short-window models + static checks.
  - `xteam audit`: After static checks pass, run 1 batched `Claude`/`GPT` diff review for security/logic.
- **Default Thinking Mode:** Always use `medium` or `low`. Never use `high` or `xhigh` unless explicitly requested.
- **0-Token Static Check First:** `Builder` must run `build`, `typecheck`, `lint`, and unit tests before any LLM `Reviewer` is called.
- **Triage:**
  - *Trivial:* Single agent (`Flash` `low`/`medium`).
  - *Standard:* `Scout` -> `Builder` (`Neon`/`Hydrogen`) -> static checks + cross-check review (`Hydrogen`/`Neon`; do **not** spawn `Claude`/`GPT`).
  - *Complex / High-Risk:* `Scout` -> `Architect` (`Orchestrator (Neon)` / `Neon` / `Argon`) -> `Builder(s)` -> post-merge build & static checks -> **at most 1 batched `Claude`/`GPT` review** -> `Tester` -> cleanup.
- **Quota Thresholds (see [model-roles.md](./references/model-roles.md) for failover matrix):**
  - `< 30%` remaining: Soft-shift routine/review work from scarce pools to short-window or separate-pool models.
  - `<= 5%` remaining: Hard failover to the designated backup role/model.
  - **Hot-Swap Before Compact:** Use `swarm(action="model")` only for short/medium workers (`<= 15–20` messages); if context is bloated, capture a 5-line summary, `kill` the worker, and start the fallback fresh.
  - **Total Exhaustion (All `<= 5%`):** Stop spawning `swarm` workers, switch to Single-Agent Ultra-Lean mode (`rg -n` + minimal line-window reads), and warn the user.

## Office Role Hierarchy

| Role | Primary Model | Fallback | Mode | Responsibility & Slash Command Hint |
| :--- | :--- | :--- | :--- | :--- |
| **Scout** | `gemini-3.8-flash` | `hydrogen-low` / `neon-low` | `low` / `medium` | Search, logs, summaries, mechanical edits. Suggest `/compact`, `/schedule`. |
| **Builder** | `gemini-4-neon` / `gemini-4-hydrogen` | `flash-medium` / `argon` | `medium` | Heavy coding, refactoring, post-merge build & tests. Suggest `/goal`, `/teamwork`. |
| **Architect** | `orchestrator-neon` / `gemini-4-neon` | `argon` / `opus` | `medium` | Critical design & blockers (summary-fed; 5h reset via Neon). Suggest `/plan`, `/grill-me`, `/learn`. |
| **Reviewer** | `hydrogen` <-> `neon` / `argon` (Std) | `claude-sonnet` / `gpt-oss` (High-Risk) | `low` / `medium` | Diff-only bug/security review after static checks pass. Suggest `/boost`. |
| **Tester** | `browser` subagent | — | `flash` | Live UI checks on verified `http://localhost:<port>`. Suggest `/browser`. |
| **Designer** | `image-generator` subagent | — | — | UI mockups, icons, banners, and diagrams. |

## Execution Guardrails

1. **Strict File Ownership & Build Lock:** Define shared types/contracts first. Assign parallel `Builder` workers (max 2–3) disjoint paths. Parallel workers write code only and must not run install or build commands; one designated `Builder` or Coordinator runs install/build after merging.
2. **2-Round Circuit Breaker:** Cap `Builder` <-> `Reviewer` fix loops at **2 iterations**. If unresolved, halt, summarize the blocker/diff, and escalate once to `Architect` or the user.
3. **Orphan Worker Cleanup:** Monitor via `swarm(action="status")`; diagnose errors before `swarm(action="kill")`, allowing at most 1 retry.
4. **Browser Precondition & Daemon Cleanup:** Start the dev server (`IsDaemon=true`) only after build passes, hand `browser` an explicit URL/port + test steps, and `manage_task` `kill` the daemon immediately after verification.

## On-Demand References

Read only when needed:
- Failover matrix, worker briefs, and subagent dispatch: [model-roles.md](./references/model-roles.md)
- Static-first workflow, batched reviews, and context hygiene: [token-saving-rules.md](./references/token-saving-rules.md)
