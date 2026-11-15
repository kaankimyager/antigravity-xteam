# antigravity-xteam

A quota-aware multi-model orchestration skill built exclusively for **Google Antigravity**.

> **Antigravity Only:** This skill relies on Antigravity's progressive skill loader (`SKILL.md` + `references/`), `swarm` worker pinning and live quota inspection, `invoke_subagent` (`browser`, `image-generator`, `research`), and `manage_task` daemon controls. It is not intended for other editors or agent frameworks.

---

## Why This Exists

Running every step of a coding task on frontier models drains weekly quotas fast—especially when Claude and GPT models share a combined pool and each newly spawned worker pays a full context/tool-schema boot cost.

`antigravity-xteam` splits the work the way a practical engineering team would:

- **High-throughput models (`Gemini 3.8 Flash`, `Gemini 4 Hydrogen`)** handle codebase search, file reads, primary implementation, and refactoring.
- **Local tooling (`build`, `typecheck`, `lint`, unit tests)** runs *before* any LLM review so syntax and type errors cost zero tokens.
- **Separate-pool or scarce models (`Gemini 4 Argon`, `Claude Sonnet`, `Claude Opus`, `GPT-OSS 120B`)** are reserved for architecture checkpoints and batched, diff-only reviews on complex or high-risk changes.
- **Default thinking mode is locked to `medium` or `low`** (`high` / `xhigh` are avoided unless you explicitly ask for them).

---

## Role Structure

| Role | Default Model | Fallback / Escalation | Responsibility |
| :--- | :--- | :--- | :--- |
| **Scout** | `gemini-3.8-flash` (`low` / `medium`) | `gemini-4-hydrogen-low` | `rg -n` searches, narrow line-range reads, summaries, routine edits. |
| **Builder** | `gemini-4-hydrogen` (`medium`) | `gemini-3.8-flash-medium` / `argon` | Primary implementation, refactoring, post-merge builds and test runs. |
| **Architect** | `gemini-4-argon` (`medium`) | `hydrogen` / `claude-opus-5-5-medium` | System design and hard blockers (fed concise summaries, never raw repo dumps). |
| **Reviewer** | `hydrogen` / `argon` (Standard) | `claude-sonnet` / `gpt-oss-120b` (High-Risk) | Diff-only bug and security review after static checks pass. |
| **Tester** | `browser` subagent (`flash`) | — | Live UI verification against a verified `http://localhost:<port>`. |
| **Designer** | `image-generator` subagent | — | UI mockups, icons, banners, and diagrams. |

---

## Built-In Guardrails

- **Live Model Resolution:** Always inspects `swarm(action="models", all=true)` before spawning workers instead of assuming hardcoded model IDs never change.
- **Two-Stage Quota Failover:**
  - `< 30%` remaining on a scarce pool shifts routine work to `Argon` or `Hydrogen`.
  - `<= 5%` triggers hard role failover across model families (and drops to **Single-Agent Ultra-Lean Mode** if all pools are exhausted).
- **Hot-Swap Before Compact:** Short worker sessions (`<= 15–20` messages) can switch models mid-flight via `swarm(action="model")`; bloated sessions are summarized in 5 lines, killed, and restarted clean so history doesn't burn the fallback's quota.
- **Strict File Ownership & Build Lock:** Parallel `Builder` workers (capped at 2–3) only touch disjoint paths after shared contracts are written, and never run `install` or `build` commands concurrently—one designated `Builder` or the coordinator builds after merging.
- **2-Round Circuit Breaker:** `Builder` <-> `Reviewer` fix loops stop after 2 rounds and escalate once rather than looping indefinitely.
- **Daemon Cleanup:** Any background dev server started with `IsDaemon=true` for `browser` testing is terminated immediately via `manage_task` once verification finishes.

---

## Installation (Antigravity)

### Option 1: Global Install (Recommended — applies to all workspaces)

**macOS / Linux:**
```bash
git clone https://github.com/kaankimyager/antigravity-xteam.git ~/.gemini/config/skills/antigravity-xteam
```

**Windows (PowerShell):**
```powershell
git clone https://github.com/kaankimyager/antigravity-xteam.git "$env:USERPROFILE\.gemini\config\skills\antigravity-xteam"
```

### Option 2: Workspace-Only Install

Copy the skill folder into your project's `.agents/skills/` directory:

```bash
git clone https://github.com/kaankimyager/antigravity-xteam.git .agents/skills/antigravity-xteam
```

---

## Usage

Once installed, Antigravity discovers `antigravity-xteam` automatically via `SKILL.md` frontmatter.

- **Recommended main chat model:** `Gemini 4 Hydrogen (Medium)` for general engineering work, or `Gemini 3.8 Flash (Medium)` for quick exploration and light edits.
- **Avoid leaving `Claude Opus` or `Claude Sonnet` selected as your main chat model**—let the skill invoke scarce models surgically only when a complex diff or blocker warrants it.
- Reference files inside `./references/` are loaded on demand only when needed to keep context usage low.

---

## Repository Layout

```text
antigravity-xteam/
├── SKILL.md                          # Core orchestrator instructions (kept lean)
├── references/
│   ├── model-roles.md                # Live matching, 5-row failover matrix, worker briefs
│   └── token-saving-rules.md         # Static-first gate, batched reviews, context hygiene
├── LICENSE
└── README.md
```

## License

MIT — see [LICENSE](./LICENSE).
