---
type: llm
focus: trace
weight: 1
---

Executed branch: every coverage row uses exactly one of the six statuses `planned`, `covered`, `open`, `unknown`, `deviated`, or `out-of-scope`; every non-covered row has a reason, and every deviated/out-of-scope row has an explicit user/product approval plus an evidence reference. No agent-declared scope reduction and no silent omission. Blocked infrastructure (documented missing fixture/URL/automation): N/A, not a pass; see the blocked-infrastructure grader. In the executed branch, planned/open/unknown rows are not counted as covered.
