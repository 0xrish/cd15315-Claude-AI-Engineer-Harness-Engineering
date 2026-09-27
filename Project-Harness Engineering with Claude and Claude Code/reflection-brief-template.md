# Reflection Brief — Harness Engineering Capstone

**Name:** Claude AI Engineer
**Date:** 2026-09-27

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001`
- OS / Python: Windows 11 / Python 3.12.10
- Approx. API spend: ~$0.21 USD ($0.1306 USD across the 8 System 1 claims in [`evidence/system-1/summary.md`](evidence/system-1/summary.md) + ~$0.08 USD for System 2 extraction and compression in [`evidence/system-2/budget.json`](evidence/system-2/budget.json); System 4 run offline with `--recorded-response` at $0.00)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → The file and function is [`claims_intake/loop.py:run()`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/loop.py#L58-L132). In this function, the harness continuously dispatches on `response.stop_reason`: when `stop_reason == "tool_use"`, it appends the assistant turn, executes each tool call via `tool_executor`, packages results into a single user turn, and `continue`s; when `stop_reason == "end_turn"`, it appends the assistant turn and returns `FinalState`. Any other value raises `UnexpectedStopReason(f"turn {turn}: unexpected stop_reason={response.stop_reason!r}")`. For `claim_03_water_damage`, our live run trace in [`evidence/system-1/traces/claim_03_water_damage.jsonl`](evidence/system-1/traces/claim_03_water_damage.jsonl) records a 7-turn sequence: `["tool_use", "tool_use", "tool_use", "tool_use", "tool_use", "tool_use", "end_turn"]`, ending only after `route_to_adjuster` was invoked on turn 6.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → [`tests/test_antipatterns.py:test_no_integer_literal_iteration_cap_in_loop`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/tests/test_antipatterns.py#L51-L67) checks that there is no artificial turn cap like `while turn < N` or `for _ in range(N)`. If the loop used an arbitrary turn cap such as 5 turns, complex claims requiring dynamic decomposition (such as `claim_03_water_damage`, which required 7 turns for policy lookup, recording incident facts, asking a clarification question, recording the cause fact, classifying the claim, assessing severity, and terminal routing) would be forcibly killed before reaching the terminal tool. Another check (`test_no_string_membership_against_text_in_loop`) prevents string-matching on completion tags like `"[DONE]"`, which would cause false completions whenever a claimant or policy document mentions the string.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → The two terminal tools [`route_to_adjuster`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/tools.py#L90-L105) and [`escalate_to_human`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/tools.py#L106-L125) both accept claim summary data and represent terminal actions. The description of `route_to_adjuster` explicitly mandates: *"TERMINAL TOOL. The agent picks this when classification confidence is >= 0.6 and severity has been assessed"*, whereas `escalate_to_human` mandates: *"TERMINAL TOOL. The agent picks this when the claim cannot be routed safely"*, requiring fields like `root_cause`, `candidate_claim_types`, and `recommended_action`. When a tool call errors, returning a structured JSON string (`{"is_error": True, "error_category": "permanent", "is_retryable": False, "message": "..."}`) enables the model on the next turn to distinguish between transient network issues and bad inputs (such as invalid `policy_id`), allowing it to adjust its parameters or escalate rather than repeatedly failing or crashing the process.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → In our live run captured in [`evidence/system-1/summary.md`](evidence/system-1/summary.md), `claim_03_water_damage` completed in **7 turns** with 1 clarification asked, consuming **25,826 input tokens** and **1,253 output tokens**, with an estimated cost of **$0.0321 USD** (elapsed time: 16.4s). Total estimated cost across all 8 processed fixtures was **$0.1306 USD**. This differs from the README sample (~$0.05 across all 8 claims combined, with claim 3 around 4 turns) because the live model dynamically decomposed the incident over 7 full turns—performing initial lookup, collecting 4 claim facts, asking a clarification question regarding the water source, ingesting the claimant's reply, classifying, assessing high severity, and finally invoking `route_to_adjuster`.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → Sourced from our live run artifact [`evidence/system-2/budget.json`](evidence/system-2/budget.json) (run `20260927-151925`), the baseline transcript is **38,708 tokens**, and the assembled context is **16,821 tokens**, yielding a **56.54% reduction**. The `active` verbatim section dominates at **15,789 tokens** (~93.86% of the assembled context). It is preserved byte-exact because the active issue (payment-method update with AVS mismatch) is currently in-flight; any loss of nuance, turn sequence, or specific customer phrasing could cause decision errors or repeat questions during live interaction.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The architectural rule enforced in [`retail_context/compressor.py:compress()`](../Engineer%20a%20Long-Conversation%20Context%20Strategy%20for%20a%20Retail%20Support%20Copilot/04-assemble-and-locate/solution/retail_context/compressor.py#L50-L75) is: **closed/resolved threads are summarized; open/active threads are preserved byte-exact**. From [`evidence/system-2/budget.json`](evidence/system-2/budget.json), the resolved segments (`resolved_refund` at 379 tokens from 12,344 input tokens, and `resolved_subscription` at 467 tokens from 11,485 input tokens) compressed ~23,800 raw tokens down to 846 tokens (~96.4% compression) while distilling key facts into the **204-token** `case_facts` block. The active segment (**15,789 tokens**) is kept 100% byte-exact as verified by [`tests/test_assemble.py:test_active_segment_byte_exact`](../Engineer%20a%20Long-Conversation%20Context%20Strategy%20for%20a%20Retail%20Support%20Copilot/04-assemble-and-locate/solution/tests/test_assemble.py#L32-L48).

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In [`evidence/system-2/eval.jsonl`](evidence/system-2/eval.jsonl), all 6 questions passed (6/6 PASS). In [`evidence/system-2/eval_control.jsonl`](evidence/system-2/eval_control.jsonl), where the `# Case Facts` block was stripped, **Question 6 regressed from PASS to FAIL**. Question 6 asks: *"What is the structured status of the payment-method update issue (use the exact status token from the case record, not a paraphrase)?"* Because this status token (`in_progress`) was established earlier and summarized away from the raw transcript, removing the `# Case Facts` block leaves the model with no source for the structured token. This empirically proves that the persistent case-facts block is load-bearing and prevents catastrophic memory loss over long multi-issue conversations.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → In [`.claude/rules/tests.md`](../Configure%20Claude%20Code%20for%20a%20Multi-Surface%20Monorepo%20Team/04-plan-mode-and-explore-decision-doc/solution/.claude/rules/tests.md#L1-L6):
   ```yaml
   ---
   description: Conventions for test files (co-located *.test.ts and *.test.tsx)
   paths:
     - "**/*.test.tsx"
     - "**/*.test.ts"
   ---
   ```
   A path-scoped glob rule matches across test files scattered throughout the entire monorepo hierarchy (e.g. `src/api/**/*.test.ts`, `src/web/**/*.test.tsx`, `packages/**/*.test.ts`). A directory-level `CLAUDE.md` is strictly bound to a single directory tree; enforcing a cross-cutting testing standard with `CLAUDE.md` would require duplicating identical instructions across dozens of subdirectories or putting them at the root where they pollute non-test files. Glob rules dynamically activate only when the model touches matching test files anywhere in the workspace.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → In [`.claude/skills/deploy-check/SKILL.md`](../Configure%20Claude%20Code%20for%20a%20Multi-Surface%20Monorepo%20Team/04-plan-mode-and-explore-decision-doc/solution/.claude/skills/deploy-check/SKILL.md#L4-L16):
   ```yaml
   context: fork
   argument-hint: "[target-branch] (defaults to main)"
   allowed-tools:
     - Read
     - Grep
     - Glob
     - Bash(git status:*)
     - Bash(git diff:*)
     - Bash(git log:*)
     - Bash(git rev-parse:*)
     - Bash(git ls-files:*)
     - Bash(gh pr view:*)
     - Bash(gh pr checks:*)
   ```
   Running in a forked sub-agent (`context: fork`) ensures that the verbose output of pre-deployment inspection (git status dumps, diff checks, file listings) runs in an isolated sub-agent context, returning only the single pass/fail summary to the main conversation. The strictly scoped `allowed-tools` allowlist restricts Bash to read-only subcommands (`git status:*`, `git diff:*`, `git log:*`, `git rev-parse:*`, `git ls-files:*`, `gh pr view:*`, `gh pr checks:*`) alongside `Read`, `Grep`, and `Glob`. This deterministically guarantees that a pre-flight deployment check cannot modify code, execute arbitrary shell scripts, or inadvertently trigger a real deployment or destructive mutation. Without the fork, megabytes of intermediate output pollute the parent context; without read-only tool scoping, an autonomous agent running deployment checks could write or mutate production state.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator (`ecommerce-team-config .`) passes all 35 tests verifying project-level vs user-level boundaries. Project-level configuration lives in version control under `.claude/` (e.g., [`.claude/commands/review.md`](../Configure%20Claude%20Code%20for%20a%20Multi-Surface%20Monorepo%20Team/04-plan-mode-and-explore-decision-doc/solution/.claude/commands/review.md) and [`.claude/rules/tests.md`](../Configure%20Claude%20Code%20for%20a%20Multi-Surface%20Monorepo%20Team/04-plan-mode-and-explore-decision-doc/solution/.claude/rules/tests.md)) to enforce unified CI commands, review criteria, and glob-scoped rules across the team. User-level configuration (such as personal `~/.claude/CLAUDE.md` or personal API keys and local editor settings) applies only to the individual engineer without polluting the shared repository.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → In [`shift_monitor/warm.py:defects_since()`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/warm.py#L75-L85), the query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, backed by index `idx_defects_ts ON defects(ts)`. In our live run with `--since 2026-04-20T00:00:00Z` against the seeded 40-defect database (`fixtures/defects.json`), the query returned 5 new defects for shift C (a slice of 5 returned vs 40 warm-tier total), as logged in `evidence/system-4/shift_output.txt`. Pushing filtering down to SQLite ensures that the LLM only receives new, actionable records within the time window rather than consuming tens of thousands of tokens loading months of historical defect records into context.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In [`shift_monitor/recovery.py`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/recovery.py#L12-L35), `STALE_RESUME_THRESHOLD_MINUTES = 30`. If a crash occurred <= 30 minutes ago, the system resumes mid-flight steps; if > 30 minutes, it initiates a fresh run with an injected summary of past completed steps. In manufacturing quality monitoring, conditions change rapidly; resuming a 2-hour-old partial prompt risks acting on outdated machine telemetry, whereas a fresh run re-queries current sensor states while retaining the high-level context of what failed. Furthermore, as implemented in [`shift_monitor/fork.py:fork_for_hypothesis()`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/fork.py) and verified by [`test_two_forks_produce_independent_scratchpads`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/tests/test_us04_fork_scratchpad.py#L50-L75), forked investigations stay isolated by creating independent scratchpad JSONL files and state deep-copies for competing hypotheses without mutating the primary shift stream; validated findings are merged back only via `merge_findings()` through append-only fsynced writes.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → Our generated [`evidence/system-4/hot_state.json`](evidence/system-4/hot_state.json) is **752 bytes**, well within the 5,000-byte budget enforced by [`shift_monitor/state.py`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/state.py#L17-L20) (`HOT_STATE_BYTE_BUDGET = 5000`, `MAX_RECENT_HASHES = 20`). For a system running 3 shifts a day, 365 days a year indefinitely (over 1,000 shifts/year), unbound state would inevitably exceed context limits and degrade prompt attention; capping hot state at ~5 KB guarantees constant-time boot and stable token costs forever.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model:** The raw inference invocation in [`retail_context/client.py:complete_with_system`](../Engineer%20a%20Long-Conversation%20Context%20Strategy%20for%20a%20Retail%20Support%20Copilot/04-assemble-and-locate/solution/retail_context/client.py#L60-L95), which formats prompts and queries the Anthropic Messages API.
    → **Harness:** The execution loop in [`claims_intake/loop.py:run()`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/loop.py#L58-L132), which wraps the model in a while-loop, dispatches tools based on `stop_reason`, enforces token budgets, and emits traces.
    → **Orchestration:** The multi-tier orchestration pipeline in [`shift_monitor/pipeline.py:run_shift()`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/pipeline.py#L90-L150), which coordinates external SQLite storage, atomic state transitions, crash-recovery manifests, and sub-agent hypothesis forks.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → In System 3, [`.claude/skills/deploy-check/SKILL.md`](../Configure%20Claude%20Code%20for%20a%20Multi-Surface%20Monorepo%20Team/04-plan-mode-and-explore-decision-doc/solution/.claude/skills/deploy-check/SKILL.md#L6-L16) deterministically enforces `allowed-tools: [Read, Grep, Glob, Bash(git ...:*), Bash(gh pr ...:*)]`, and System 1 deterministically short-circuits on `session.terminal_called` in [`claims_intake/tools.py`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/tools.py#L190-L210). Conversely, whether a water damage claim classifies as `property_damage` vs `liability` is guided by the prompt in [`claims_intake/system_prompt.py`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/system_prompt.py). Deterministic code is required for safety boundaries, permissions, and invariants; prompt guidance is appropriate for semantic interpretation, ambiguity detection, and classification nuance.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages **intra-session context** during a 48-turn conversation by compressing resolved threads to 846 tokens and distilling case facts into a 204-token header, achieving a **56.54% token reduction** (reducing 38,708 baseline tokens down to 16,821 assembled tokens, with 15,789 active tokens kept verbatim) as recorded in [`evidence/system-2/budget.json`](evidence/system-2/budget.json). System 4 manages **cross-session context** across indefinite 8-hour shifts by offloading history to SQLite and maintaining a compact **752-byte** `hot_state.json` ([`evidence/system-4/hot_state.json`](evidence/system-4/hot_state.json)), pre-filtering with index `idx_defects_ts` to slice 5 new defects out of 40 warm-tier total ([`evidence/system-4/shift_output.txt`](evidence/system-4/shift_output.txt)). Both adhere to the principle of keeping LLM context small and high-signal, but System 2 compresses in-memory narrative via LLM summarization, whereas System 4 pushes state down to SQLite and Pydantic-validated JSON.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → [`tests/test_us03_crash_recovery.py:test_mid_write_read_reveals_prior_complete_lines`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/tests/test_us03_crash_recovery.py#L30-L55) verifies that a crash mid-line during an append-step write leaves prior JSONL records intact and loadable without corrupting the file. A single normal shift run only executes the happy path where no crash occurs; only an automated fault-injection test guarantees that unexpected host reboots or SIGKILL events will not corrupt the manifest on disk.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → For System 1 (Insurance Claims Intake), an unconstrained agent could loop infinitely making costly API calls or misroute valid claims. The blast radius is constrained by two hard enforcement points: [`claims_intake/budget.py:Budget`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/budget.py#L25-L65), which acts as a hard token/wall-clock kill switch raising `BudgetExceeded` if exceeded, and the terminal tool check `session.terminal_called` in [`claims_intake/tools.py`](../Build%20a%20Claims%20Intake%20Agent%20with%20a%20stop_reason-Driven%20Loop/exercises/03-dynamic-decomposition/solution/claims_intake/tools.py#L190-L210), which guarantees that a claim can only be dispatched once to an adjuster or human queue.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → On Windows, the original git index contained a path with trailing whitespace (`Project-Harness Engineering with Claude and Claude Code /`), causing git index operations to raise `fatal: make_cache_entry failed for path ...` under NTFS path protection. This was resolved by setting `git config core.protectNTFS false` and handling Windows-compatible directory paths. Additionally, Pydantic v2 validation rules required using `Field(max_length=20)` rather than deprecated v1 parameters in [`shift_monitor/state.py`](../Build%20a%20Multi-Shift%20Quality%20Monitoring%20System%20with%20Claude%20Orchestration/04-fork-scratchpad/solution/shift_monitor/state.py).

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → In System 2, context compression relies on a dedicated LLM call per resolved segment during context assembly. For high-volume customer support systems, running synchronous LLM compression adds latency and cost to the user's turn; I would transition this to an asynchronous background worker that automatically compresses and updates the case-facts block as soon as an issue status changes to `resolved`, so the interactive copilot turn only ever performs lightweight string concatenation.
