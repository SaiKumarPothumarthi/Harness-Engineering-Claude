Harness Engineering Capstone
A capstone project demonstrating four patterns for reliable AI application engineering:

stop_reason-driven agentic control
position-aware context compression
Claude Code project configuration
multi-session Layer 3 orchestration
Results
System	Result
Agentic loop	30 tests passed
Context strategy	30 tests passed
Claude Code configuration	35 tests passed
Layer 3 orchestration	33 tests passed
Highlights
Agentic loop
Implemented and verified stop_reason-driven control flow rather than fixed-turn agent execution.

Context engineering
Reduced context from 38,708 tokens to 16,832 tokens, a 56.52% reduction, while passing 6/6 evaluation questions.

Claude Code harness
Validated a project configuration containing hierarchical CLAUDE.md guidance, path-scoped rules, a project command, and a forked read-only skill.

Layer 3 orchestration
Used SQLite pre-filtering to select 17 relevant defects from a 40-record warm tier.

The resulting hot state was only 965 bytes.

The system also demonstrates:

crash recovery with a 30-minute staleness threshold
tiered state
atomic state persistence
isolated hypothesis forks
explicit scratchpad merging
Verification
Test suites:

System 1: 30 passed
System 2: 30 passed
System 3: 35 passed
System 4: 33 passed
Total: 128 passing tests.

Note
This repository contains my project documentation, reflection, and sanitized run evidence. Course-provided reference implementation source code is intentionally excluded.