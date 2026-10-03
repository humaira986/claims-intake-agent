# Reflection Brief — Harness Engineering Capstone

**Name:** Humaira

**Date:** 2026-10-03

Every answer below cites an artifact from my own runs, including run IDs, file paths, token counts, claim outcomes, or test counts.

---

## Environment

- Model(s): Claude model accessed through the Vocareum Anthropic endpoint used by the course.

- OS / Python: Linux environment in Vocareum; Python 3.13.0 for the capstone workspace.

- Approx. API spend: Approximately $0.1195 for the System 1 claims run. System 2 used Anthropic `messages.count_tokens` for model-authoritative token budgeting.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

→ In `evidence/system1_agentic_loop/runs/20261003_034218/traces/claim_04_neighbor_injury.jsonl`, the `stop_reason` sequence is `tool_use, tool_use, tool_use, tool_use, tool_use, end_turn`. The continue-vs-stop decision is implemented in `claims_intake/loop.py::run`. The loop continues when the response has `stop_reason == "tool_use"` and returns when the stop reason is `end_turn`; an unexpected stop reason is handled as an unexpected stop. This makes the model's `stop_reason` the loop-control signal rather than parsing the response text or using a fixed iteration count.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

→ One anti-pattern checked by `tests/test_antipatterns.py` is an integer-literal iteration cap, covered by `test_no_integer_literal_iteration_cap_in_loop`. My `claim_04_neighbor_injury` run needed 6 turns, with five `tool_use` turns before the final `end_turn`, so a small fixed cap could have stopped the claim before the agent finished gathering and recording the required information. The same test suite also verifies that `stop_reason` controls the loop. The actual run is recorded under `evidence/system1_agentic_loop/runs/20261003_034218/`.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

→ Two related tools in `claims_intake/tools.py` are `record_claim_fact` and `request_clarification`. Their descriptions distinguish recording a verified claim fact from asking the claimant for missing information, so the model has an explicit purpose for each tool instead of treating both as generic text operations. The structured tool error reports the failure in a machine-readable form, allowing the agent to understand what was invalid and take another tool action instead of treating an arbitrary error string as a final answer. These tool behaviors were exercised by the System 1 tests and the `20261003_034218` run.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

→ `claim_04_neighbor_injury` completed in 6 turns, including 2 clarification requests, with an estimated cost of approximately `$0.0296`. The result is recorded in `evidence/system1_agentic_loop/runs/20261003_034218/summary.md` and its trace is `traces/claim_04_neighbor_injury.jsonl`. My result does not need to match the README sample exactly because the README example is illustrative while my run used the actual claim fixture and live Vocareum API execution. The extra turns were caused by the information needed for this specific claim, including the two clarification steps.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

→ `evidence/system2_context_strategy/budget.json` records 38,708 baseline tokens and 16,972 assembled tokens, giving a 56.15% reduction. The `active` section dominates the assembled context at 15,789 tokens. I kept the active section intact because it represents the current working state and exact details needed for the next interaction, where summarizing could remove information that still affects the answer. The run also recorded a 6/6 pass rate in `eval.jsonl`.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

→ The context strategy summarizes resolved or historical material while preserving the current active material and small factual blocks needed for exact retrieval. In `budget.json`, `case_facts` was 204 tokens, `resolved_refund` was 419 tokens, `resolved_subscription` was 578 tokens, and `active` was 15,789 tokens in the assembled context. The resolved sections are compressed because their information has already been settled, while the active section is preserved because it contains information still relevant to the current task. This produced 16,972 assembled tokens from a 38,708-token baseline.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

→ `evidence/system2_context_strategy/eval.jsonl` passed all 6 evaluation questions. In `eval_control.jsonl`, Q1 unexpectedly passed while Q6 failed as expected. Q6 concerned the `in_progress` state, showing that removing the intended context can make an answer fail even though the normal assembled context answers all six questions. The control result therefore demonstrates that the context assembly is preserving information that matters for some questions rather than merely reducing token count.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

→ In `.claude/rules/react.md`, the path-scoped frontmatter contains:
`paths:
  - "src/components/**/*"
  - "src/pages/**/*"`
This is better than putting the convention in a directory-level `CLAUDE.md` because the rule is automatically associated with the files where the React convention applies, even when those files are spread across the project. It avoids applying React-specific guidance to unrelated files. The configuration was validated by `evidence/system3_claude_config/validator_output.txt`, which reports `OK`.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

→ The deploy skill in `.claude/skills/deploy-check/SKILL.md` uses `context: fork` and an allowed-tool list containing `Read`, `Grep`, `Glob`, `Bash(git status:*)`, `Bash(git diff:*)`, and `Bash(git log:*)`. The fork keeps the detailed discovery work out of the main conversation context, while the read-only allowlist prevents the skill from modifying files, pushing commits, or deploying. Without the fork, verbose investigation could consume the main context; without the read-only restrictions, a validation command could accidentally make changes or perform deployment actions. The configuration passed the System 3 suite with 35 tests, recorded in `evidence/system3_claude_config/pytest_S3.log`.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

→ `evidence/system3_claude_config/validator_output.txt` reports the configuration as valid with `OK`. A project-level example is the repository's `CLAUDE.md` and `.claude/rules/react.md`, which apply to this project. A user-level example is the user Claude configuration under `~/.claude/skills/deploy-check-strict/`, which is outside the project directory. Keeping these scopes separate prevents personal/global preferences from being confused with the project's required engineering rules.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

→ The warm database contained 40 defects, but the SQL-filtered query for Shift C returned only 2 new defects since `2026-04-26T00:00:00Z`. The query is implemented through `WarmStore.defects_since("2026-04-26T00:00:00Z", 50)` and filters the `defects` table by timestamp; the query plan recorded in `evidence/system4_orchestration/orchestration-evidence.md` is `SEARCH defects USING INDEX idx_defects_ts (ts>?)`. The model therefore receives the small relevant defect slice instead of the entire 40-record history. This pushes deterministic filtering into the database layer before model reasoning.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

→ The recovery logic in `recovery.py` uses a 30-minute staleness threshold. A recent incomplete state within the threshold can be resumed, while a stale state, such as the recorded 31-minute-old case, starts fresh; empty or already-complete state also starts fresh. A fresh start with an injected summary can be more reliable because it avoids depending on potentially stale or partially corrupted transient state while still giving the new session the important prior information. These recovery cases are documented in `evidence/system4_orchestration/orchestration-evidence.md`.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

→ `evidence/system4_orchestration/hot_state_size.txt` records the hot state at 680 bytes. Keeping the state this small makes each shift cheap to load and reduces the amount of transient information that can accumulate over repeated runs. Because the system may execute once per shift indefinitely, an unbounded state file would eventually increase context, recovery, and storage costs. The small-state design therefore keeps the cross-shift state bounded.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.

→ **Model:** `claims_intake/loop.py::run` in the System 1 solution is the model-driven decision layer; the trace `evidence/system1_agentic_loop/runs/20261003_034218/traces/claim_04_neighbor_injury.jsonl` shows the model's `tool_use` and final `end_turn` decisions.

→ **Harness:** `CLAUDE.md` and the `.claude/rules/` and `.claude/skills/` files in `evidence/system3_claude_config/` define the project-specific working rules, path-scoped conventions, and safe deployment-check behavior.

→ **Orchestration:** `recovery.py`, `fork.py`, `data/warm.sqlite`, `data/hot_state.json`, and `data/shift_scratchpad.jsonl` in System 4 manage filtering, recovery, state, and isolated shift execution. Together these artifacts separate model reasoning, developer-tool constraints, and operational state management.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code and one guided by prompt. When is each right?

→ A deterministic behavior is the read-only `allowed-tools` list in `.claude/skills/deploy-check/SKILL.md`, which limits the forked skill to reading and selected Git inspection commands. A prompt-guided behavior is the skill instruction that it should perform a read-only pre-deployment check and return a single structured summary instead of changing the project. Deterministic enforcement is appropriate for safety boundaries and invariants that must not be violated. Prompt guidance is appropriate for flexible behavior, explanation, and workflow choices where strict machine enforcement is unnecessary.

16. **Context, two faces.** Compare context management in System 2 and System 4 with cited numbers from both. Same principle, different mechanism — how?

→ System 2 manages context within a session by reducing 38,708 baseline tokens to 16,972 assembled tokens, a 56.15% reduction, while preserving the 15,789-token active section. System 4 manages context across shifts by keeping the hot state at only 680 bytes and loading only 2 relevant defects from a 40-defect warm database for the recorded Shift C run. Both systems apply the same principle: keep the information needed for the next decision and avoid carrying unnecessary history. System 2 uses context assembly and summarization, while System 4 uses database filtering, bounded state, and recovery rules.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

→ The System 1 test `test_no_integer_literal_iteration_cap_in_loop` guarantees that the loop does not contain a fixed integer iteration cap. A single successful run, such as `claim_04_neighbor_injury` completing in 6 turns, would not prove that another claim could not be silently cut off by a hidden cap. The test therefore checks the implementation invariant rather than only the observed outcome of one fixture. This matters before shipping because different claims can require different numbers of tool interactions.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

→ For System 4, a failure could affect the shift's defect selection, hot state, scratchpad, and recovery behavior, potentially influencing what defects are presented for action. The main enforcement points are the SQL timestamp filter, the bounded hot-state design, the recovery threshold in `recovery.py`, and the fork isolation in `fork.py`; the recorded evidence shows `base unchanged: True` and `forks isolated: True`. The practical kill switch is to stop the shift process and avoid promoting its generated state or recommendations until the failing run is inspected. The isolated fork design limits the effect of one shift execution on the base state.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it.

→ The first System 1 API run failed because `anthropic==0.39.0` was incompatible with the installed `httpx==0.28.1`; the error was `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'`. I fixed the environment by using `httpx==0.27.2`, after which the actual claims run completed successfully. The final run is recorded under `evidence/system1_agentic_loop/runs/20261003_034218/`, and the System 1 test log shows 29 passing tests. This was an environment dependency issue rather than a failure of the loop implementation.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

→ I would add a stronger post-run evidence validator that checks the required reflection fields and generated artifacts before submission. The implementation and tests were passing, but the earlier review showed that the project could still fail evaluation because the reflection did not explicitly connect the required behaviors to named files, numbers, and artifacts. The System 2 budget/evaluation artifacts and System 4 orchestration evidence already provide strong machine-readable evidence, so a final validator could automatically check that the reflection references those artifacts. This would reduce the risk of a technically correct implementation being rejected because its evidence is incomplete.