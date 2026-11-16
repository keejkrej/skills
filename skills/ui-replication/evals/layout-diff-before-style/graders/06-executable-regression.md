---
type: llm
focus: trace
weight: 1
---

Executed branch (candidate and rendered extraction available): the run creates at least one framework-native executable regression check for the dialog using the repo's existing test tooling, derived from the target spec rather than from candidate constants or an auto-accepted snapshot. The test path and exact run command are reported in `.agents/ui-replication/diff/gate-report.md`. The check reads the rendered runtime (DOM/accessibility/geometry), asserts a nonempty selection or expected node count, reuses the neutral spec's canonicalization/tolerance configuration, and checks at least the structural assertions the case requires (surface presence, hierarchy/parent/sibling order, roles/names/text). Failures are fixed and the affected checks and actual spec are re-run before style polish or final screenshots; unexecuted/failing required checks block the owning gate. Blocked infrastructure (documented missing fixture/URL/automation): N/A, not a pass; see the blocked-infrastructure grader. A test that selects zero nodes, copies candidate values, or loosens tolerance to pass does not satisfy the executed branch.
