# Reflection Brief — Harness Engineering Capstone

**Name: Sai Kumar Pothumarthi**
**Date: 16th November, 2026**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):Claude-haiku-4-5-20251001
- OS / Python: Pyhton 3.13.18
- Approx. API spend:$0.3165 

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   In my System 1 run 20261116_090952, runs/20261116_090952/traces/claim_01_kitchen_fire.jsonl recorded the stop_reason sequence tool_use → end_turn. The continue-versus-stop decision is made in claims_intake/loop.py by the run() function. When response.stop_reason == "tool_use", run() executes the requested tools, appends their results to the working messages, and continues; when response.stop_reason == "end_turn", it returns the FinalState. Any other stop reason raises UnexpectedStopReason, so control flow depends on the API stop signal instead of model text or a fixed turn count.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   One anti-pattern checked by tests/test_antipatterns.py is an integer-literal iteration cap such as for _ in range(<fixed number>) or while turn < <fixed number> as the primary termination mechanism. The implementation instead uses stop_reason for termination and calls budget.check() as its safety mechanism. My run demonstrates why a fixed turn count is inappropriate: claim_01_kitchen_fire completed in 2 turns, while claim_02_stolen_bike required 5 turns (runs/20261116_090952/summary.md). A fixed cap could terminate a longer claim before its necessary tool interactions were complete; the intended behavior is also verified by the 30 passing System 1 tests in evidence/system1/tests.txt.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   route_to_adjuster and escalate_to_human overlap because both are terminal tools that can end claim processing, but their descriptions establish different preconditions. route_to_adjuster is for a classified claim with confidence of at least 0.6 after severity assessment, while escalate_to_human is for low confidence, unresolved ambiguity, missing critical facts, or policy disputes (claims_intake/tools.py). Tool failures use the structured Graceful Tool Failure fields is_error, error_category, is_retryable, and message; the system prompt tells the model to read the message and adapt rather than retry blindly. My run did not trigger a structured tool error, but this contract allows the agent to distinguish retryable transient failures from permanent invalid requests, which a generic error string would not communicate explicitly. 

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   In run 20261116_090952, claim_02_stolen_bike completed in 5 turns, using 17,312 input tokens and 836 output tokens at an estimated cost of $0.0215 (runs/20261116_090952/summary.md). The complete eight-fixture run cost $0.1100, compared with the README sample estimate of approximately $0.05 on Haiku 4.5, so my observed total was about $0.06 higher and roughly 2.2 times the sample estimate. The README value is an estimate, and actual executions can vary in model turns and tool interactions; my claims ranged from 2 to 5 turns. All eight fixtures nevertheless terminated with either a routed or escalated outcome, and all 30 System 1 tests passed.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   In my System 2 run 20261116-094926, budget.json shows a baseline of 38,708 tokens and an assembled context of 16,832 tokens, a 56.52% reduction. The active payment-method update dominates the assembled context at 15,789 tokens, compared with 204 tokens for case facts, 389 for the resolved refund, and 468 for the resolved subscription. retail_context/compressor.py deliberately preserves the active segment byte-exact instead of summarizing it. This retains the current turn-by-turn interaction and its precise troubleshooting details while aggressively compressing already-resolved issues.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   The rule in retail_context/compressor.py is to summarize segments whose status is resolved and preserve the active segment byte-exact. In my run, the resolved refund used 12,334 input tokens during compression and became a 389-token assembled section, while the resolved subscription used 11,475 input tokens and became a 468-token assembled section. The active payment-method update was never summarized and remained a 15,789-token section. retail_context/assemble.py then places the 204-token Case Facts block at the top, compact resolved summaries in the middle, and the byte-exact active issue at the bottom.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   My engineered context passed 6/6 evaluation questions in runs/20261116-094926/eval.jsonl. In the case-facts-stripped control, Q6 regressed: it asked for the exact structured status of the payment-method update, whose expected token was in_progress, but the control response said no structured status token was available. Q1 still passed in the control, so this was a specific information-loss regression rather than a general failure of the compressed context. This demonstrates that the 204-token Case Facts block preserves exact transactional values that can otherwise disappear from compressed narrative. 

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   In .claude/rules/react.md, the path-scoped frontmatter specifies paths: with "src/components/**/*" and "src/pages/**/*". These patterns activate the React conventions only for matching component and page files. This is better than maintaining separate directory-level CLAUDE.md files because one rule can cover related files across multiple repository locations without duplicating instructions. The configuration was verified by the validator returning OK and by all 35 System 3 tests passing.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   .claude/skills/deploy-check/SKILL.md declares context: fork and an allowed-tools list containing Read, Grep, Glob, and restricted read-oriented Git/GitHub commands such as Bash(git status:*), Bash(git diff:*), and Bash(gh pr checks:*). Running in a fork keeps verbose file discovery, diff inspection, and validation output outside the main session while returning only a structured pass/fail summary. The read-only allowlist prevents the skill from pushing, deploying, modifying files, or running migrations. Without the fork, intermediate investigation output would consume main-session context; without the read-only restriction, a validation skill could have an unnecessarily large mutation blast radius. 

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    The validator returned OK, and System 3 completed with 35 passing tests. A project-level example is .claude/commands/review.md, which lives inside the repository so the whole team receives the same /review workflow. The root CLAUDE.md contrasts this with user-level configuration under ~/.claude/, such as ~/.claude/commands/review-strict.md or ~/.claude/skills/deploy-check-strict/, which stays on the user's machine and is not committed for teammates. This separation keeps shared engineering policy version-controlled while supporting personal customization.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    My Shift C run returned 17 new defects from 40 defects seeded into the warm-tier SQLite store, as shown in evidence/system4/shift-output.txt. The indexed query is WarmStore.defects_since() in shift_monitor/warm.py, which executes SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?; the schema provides the idx_defects_ts index on ts. gather_new_defects() in shift_monitor/pipeline.py passes directly to this SQL query instead of loading the full table for Python-side filtering. The model therefore receives only the 17 defects relevant to the requested time range instead of the complete warm-tier history.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
     shift_monitor/recovery.py implements the resume-versus-fresh decision in decide() with STALE_RESUME_THRESHOLD_MINUTES = 30. If the manifest contains incomplete work and the latest step is no more than 30 minutes old, it returns resume; if there are no steps, the manifest is complete, or the last step is older than 30 minutes, it returns fresh. The System 4 tests verify the boundary directly: 29 and 30 minutes resume, whereas 31 minutes starts fresh, and all 33 System 4 tests passed. A fresh session with previously captured findings injected as a summary avoids continuing stale conversational assumptions while preserving durable progress.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    After my Shift C run, data/hot_state.json was only 965 bytes, recorded in evidence/system4/hot_state_size.txt. It contains recent defect hashes, the current shift summary, active alerts, and threshold statuses without copying the complete warm-tier history into active state. Keeping this state bounded matters because the orchestrator starts a fresh session for each shift and is intended to operate indefinitely; otherwise historical material would progressively increase context and processing overhead. The 33 passing tests additionally verify the hot-state size budget, a maximum of 20 recent hashes, atomic writes, and tiered-storage behavior. 
    System 4 also isolates hypothesis investigations through fork_for_hypothesis() in shift_monitor/fork.py. The function creates a separate hypothesis directory and uses shutil.copyfile() to copy the main hot_state.json, so work in the fork and its independent scratchpad does not mutate the main hot state; findings are brought back explicitly using merge_findings(). This isolation is verified by test_fork_for_hypothesis_copies_state_without_mutating_base, which confirms that the fork receives an identical copy while the base state remains byte-for-byte unchanged, and this test passed as part of my 33-test System 4 suite.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    Model: System 1's runs/20261116_090952/traces/claim_01_kitchen_fire.jsonl shows the model/API interaction: the model returns tool_use, requests tools, and eventually returns end_turn. This layer produces reasoning-dependent actions and the completion signal. 
    
    Harness: System 3's .claude/commands/review.md, .claude/rules/react.md, and .claude/skills/deploy-check/SKILL.md form the harness around model behavior through project instructions, path-scoped conventions, commands, isolation, and tool restrictions. The validator returned OK and all 35 System 3 tests passed.
    
    Orchestration: System 4's data/hot_state.json, data/warm.sqlite, and data/shift_scratchpad.jsonl manage work across shifts rather than within one model interaction. My Shift C run filtered 40 warm-tier defects to 17 relevant defects while maintaining only 965 bytes of hot state.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    System 3 provides a deterministic boundary through the allowed-tools list in .claude/skills/deploy-check/SKILL.md, which restricts the skill to read-oriented operations. In contrast, System 1 uses its system prompt to guide the model on when to classify, assess severity, route, or escalate a claim, while claims_intake/loop.py deterministically handles the returned stop_reason. Deterministic enforcement is appropriate for hard invariants such as mutation permissions, termination signals, state writes, and size budgets because those should not depend on model judgment. Prompt guidance is appropriate when behavior requires semantic judgment, such as determining whether claim evidence is sufficiently confident to route or requires human escalation. 

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
     System 2 manages context inside one long conversation: runs/20261116-094926/budget.json shows that my context was reduced from 38,708 to 16,832 tokens, a 56.52% reduction, while eval.jsonl still passed 6/6 questions. It achieves this by summarizing resolved issues while preserving the 15,789-token active segment byte-exact and retaining a compact 204-token Case Facts block. System 4 applies the same principle across sessions: SQLite filtered 40 warm-tier defects to 17 relevant defects, and hot_state.json remained only 965 bytes. Both systems minimize what occupies active model context, but System 2 uses position-aware compression within a session, while System 4 uses tiered storage, SQL pre-filtering, and compact state across independent sessions.


17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    System 4's test_hotstate_atomic_write verifies atomic persistence of hot state, while its crash-recovery tests verify behavior at the 30-minute staleness boundary, including resume at 30 minutes and fresh at 31 minutes. My successful Shift C run could not prove either property because it did not crash during a write or exercise both recovery paths. The System 4 suite passed all 33 tests, providing evidence for these abnormal paths in addition to the successful run. This matters before shipping because a long-running orchestrator must remain correct during crashes and interrupted writes, not only during its happy path.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
     System 3's deploy-check skill deliberately has a limited blast radius. .claude/skills/deploy-check/SKILL.md uses context: fork and restricts allowed-tools to Read, Grep, Glob, and read-oriented Git/GitHub operations; the skill explicitly does not push, deploy, modify files, or run migrations. If its analysis is wrong, the primary consequence is an incorrect pass/fail recommendation rather than direct repository modification or deployment. The read-only tool allowlist is the enforcement point and effective kill switch because the skill lacks mutation capabilities that could turn an incorrect analysis into a destructive action. 

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    While preparing System 4's warm SQLite tier, my first seed command failed with NameError: name 'path' is not defined because I used WarmStore(path('data/warm.sqlite')) instead of the imported Path class. I corrected it to WarmStore(Path('data/warm.sqlite')), after which the fixture data loaded successfully. I also initially supplied an unsupported --sqlite argument to shift_monitor run-shift; removing that argument allowed Shift C to complete with 17 new defects. After these corrections, all 33 System 4 tests passed and hot_state.json was successfully generated at 965 bytes. 

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
     I would add first-class observability for context and state-budget decisions rather than change the core tiered architecture. My System 2 run reduced context from 38,708 to 16,832 tokens (56.52%), while System 4 filtered 40 defects to 17 and maintained a 965-byte hot state; these measurements were useful but had to be inspected from separate artifacts. I would emit structured metrics for compression ratio, per-section token counts, SQL filter ratio, hot-state size, and recovery decisions on every run. This would preserve the existing architecture while making gradual context growth or state-budget regressions visible before they affect model quality, reliability, or operating cost.
