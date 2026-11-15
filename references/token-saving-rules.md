# Token-Saving Rules for Antigravity

Apply these practices whenever `antigravity-xteam` is active to conserve both frequent-reset and scarce weekly quotas.

## 1. Static Checks Before LLM Review (0-Token Gate)
- Always run `build`, `typecheck`, `lint`, and unit tests before invoking any `Reviewer` worker.
- Do not spend LLM review tokens on syntax, import, or type errors that local tooling catches for free.

## 2. Targeted Code Reading
- Never read entire large files blindly. Locate symbols with `rg -n` / `grep` first, then read only the relevant 50–150 line window (`StartLine` / `EndLine`).
- Do not re-read lines already in context unless modified.

## 3. Combined Pool Discipline & Batched Reviews
- Remember that `Claude` and `GPT` models often share a combined quota pool, while `Argon` may hold a separate weekly pool—check `swarm(action="models", all=true)`.
- Never spawn `Claude` or `GPT` for `Standard` tasks, trivial checks, or multi-turn back-and-forth. Batch all changed diffs into **one** concise review prompt (max 1 scarce-pool call per complex task).

## 4. 2-Round Circuit Breaker & Ultra-Lean Fallback
- Limit `Builder` <-> `Reviewer` fix cycles to **2 rounds**. If defects remain, halt and escalate once to `Architect` or the user.
- If all pools fall to `<= 5%`, stop spawning `swarm` workers and operate in **Single-Agent Ultra-Lean Mode** (`rg -n` + minimal line-window reads) while warning the user.

## 5. Context Hygiene
- Keep worker briefs terse (Goal, Scope, Constraints, Done Criteria).
- Recommend `/compact` at phase transitions (Plan -> Build -> Verify) and `kill` completed `swarm` workers promptly.
