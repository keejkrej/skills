---
type: llm
focus: trace
weight: 1
---

Executed branch (prerequisites available at start): the trace shows the computed-style and token gates running in order. On success, fixes/re-extraction and gate 6 state/behavior checks complete before gate 7 starts. On a later code-gate failure, work proceeds up to that failure and gate 7 is not run as a pass. Blocked-at-start infrastructure (fixture, URL, or automation missing before work begins): N/A, not a pass; the early-block path is judged by the blocked-infrastructure grader. Do not allow gate 7 to run ahead of gate 6.
