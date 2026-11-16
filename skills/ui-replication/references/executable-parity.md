# Executable parity checks

The neutral layout/token/states artifacts remain the canonical, portable expectations and the review/diff evidence. When the consuming stack supports it, also project those expectations into the repository's own language/framework test suite. These tests are an executable projection of Spec A, not a second source of truth and not a replacement for cross-language comparability.

Strongly prefer this projection by default when the repo already has a test runner and a way to render the UI. Do not install a framework, browser harness, or dependency the project does not already use, and do not add a package manifest for this skill.

## Stack choices

Adapt to the existing repo tooling; these are examples, not requirements:

- **Web:** DOM tests plus the project's runner (Vitest/Jest/`node:test`), with Playwright/Cypress for routes, dialogs, real geometry, and computed styles. React/Vue/Svelte Testing Library is useful for structure, roles, names, and state transitions, but jsdom-style environments cannot validate real browser layout or computed geometry.
- **Swift/macOS/iOS:** XCTest plus the platform accessibility/UI test APIs for hierarchy, labels, reachability, and geometry where exposed.
- **Android:** unit tests for pure mapping/model logic and instrumented UI tests for rendering, reachability, roles, and measured bounds.
- **Flutter:** widget tests for structure/state and golden tests as visual tripwires; real flow/geometry coverage still needs the widget/runtime render, not source constants.
- **Native equivalents:** use the platform's UI/accessibility test framework and the repository's existing runner.

## Derivation rules

- **Spec A -> test expectations.** Expected values come from independently captured reference evidence (live reference or bounded screenshot estimate), never from copying the candidate implementation or an auto-updated snapshot. A snapshot update without evidence/spec review is not parity work.
- **Candidate render -> actual assertions.** Read values from the rendered runtime (DOM/accessibility/computed style/geometry APIs), not implementation constants or config that the test merely re-reads.
- **Same canonicalization and tolerance.** Reuse the neutral spec's units, enum mapping, color representation, box arrays, null handling, and tolerance config. Do not hard-code a second set of diverging expected numbers. Where practical, generate or load fixtures from the neutral specs and compare through a small shared adapter.
- **Explicit identity mapping.** Map semantic node keys to stable test IDs, accessibility names/roles, or an explicit selector adapter. Test IDs may help selection but do not replace parent/order/structure checks.
- **Non-empty selection.** A test that selects zero nodes must fail. Assert a count or coverage count; never let an empty query pass vacuously.
- **Estimate discipline.** Match the spec's confidence: exact assertions for observable structure/content, estimate-bounded assertions for uncertain numeric values. A required check skipped because the value is unbounded/`unknown` is an unexecuted required check: it blocks its owning code gate and gate 7/completion. Only optional or not-yet-required checks may be skipped and listed as limitations. Do not turn a low-confidence visual guess into an exact test constant.
- **No universal loosening.** Do not widen tolerances just to make a suite green. Tighten or document per the neutral tolerance config.

## Coverage by gate

Order the suite incrementally and map failures back to the owning gate:

1. Surface presence and route/dialog reachability (gate 1).
2. Hierarchy, parent/child, sibling order, roles, names, text, assets, icon identity (gates 2 and 4).
3. Bounds, flow, spacing, padding/margin, border width (gate 3).
4. Computed typography, colors, border style, radius, shadows (gate 5).
5. State variants, interactions, responsive viewports, and behavior (gate 6).

Deterministic checks may cover any of these that are measurable in the stack. Do not assert browser layout behavior that the environment cannot render (for example real geometry in a DOM-only/jsdom test).

## Limits

- Snapshot tests are optional regression tripwires, not target parity: they capture current output and can be blindly updated. Pair them with explicit assertions derived from Spec A.
- Golden/screenshot tests do not replace code gates or the mandatory final gate-7 computer-use/static-capture comparison.
- A green suite does not by itself declare completion; parity still requires the ordered gates and final visual gate.
- Red, skipped, or unrun required tests stay red/skipped/unrun. They do not pass, and an approved deviation cannot convert them into passing evidence; an exact known mismatch may be recorded under the separate accepted-deviation process, but the required check itself is not reported as passing.

## Placement and reporting

Put tests where the consuming repository normally keeps framework tests, not under `.agents/`. Record in `.agents/ui-replication/diff/gate-report.md`:

- Test file paths and the exact run command.
- Environment, renderer/browser, viewports, and test counts.
- Failures and their owning gate.
- Timestamp and commit/revision where available.
- Which checks are unexecuted and why.

After every implementation fix, update or regenerate affected fixtures/tests, rerun them, and re-extract the actual spec. A required test that is failing or unexecuted blocks its owning code gate and gate 7. Report unrun tests honestly rather than treating them as passes. CI may run the deterministic suite; gate 7 remains the final release/completion gate.

## Illustrative shape (not runnable)

```
# Pseudo-adapter: load neutral target spec, read rendered candidate, compare with shared normalization.
target = load_spec("target-layout/settings/default@1280x800.yaml")
render_candidate("/settings")
nodes = select_by_semantic_keys(target.nodes)     # explicit key -> test id/role adapter
assert nodes.count == expected_count and nodes.count > 0

for node in nodes:
    actual = measure(node)                        # runtime DOM/accessibility/geometry
    compare_structure(node, actual)               # gate 2/4, exact when observable
    compare_bounds(node, actual, shared_tolerance) # gate 3, estimate-bounded if target is estimated
    compare_style(node, actual, shared_canonicalizer) # gate 5
```

Keep the adapter reusable and the per-surface tests small. Surface/structure checks come first; geometry, style, and state assertions are added once the structure is stable.
