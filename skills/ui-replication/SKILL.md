---
name: ui-replication
description: >-
  Recreate a reference UI faithfully: clone a reference app or site, turn
  screenshots into code, hit visual or pixel parity, match an existing
  interface, or audit an implementation against its reference. Use this
  whenever an existing UI is the spec, including settings panels, nested
  dialogs, responsive states, screenshot-to-code, 1:1 recreation, and
  verifying a clone before calling it done. Not for greenfield design with no
  reference. Enforces evidence-driven target/actual layout specs, explicit
  field-level diffs, and a final computer-use screenshot gate.
---

# UI replication

Approximating the main window and improvising the rest is the failure mode this skill exists to prevent. Fidelity is established by evidence, not by how finished the result looks. You transcribe the reference into a neutral layout spec before building, then independently extract the rendered result into the same spec and diff them field by field. Screenshots come last, after code parity, and neither gate alone means done.

Evidence sources — screenshots, source code, and a live or inspectable reference — are useful in any combination. Use whatever subset exists, and record each unavailable source and the resulting uncertainty explicitly rather than guessing. A completed clone still requires the final screenshot gate (7); the other sources support the work leading up to it.

## Outcome

A run is complete only when every in-scope surface/state/viewport row passes the code gates (0–6) and the final screenshot gate (7), or has a narrow deviation explicitly approved by the user/product for that exact item. "Run ended", "report complete", "blocked", and "unknown" are honest states of a run, but none of them is a passed gate or a completed replication. "Close enough", a single similarity percentage, or a clean screenshot with no code diff is not completion.

## Evidence rules

For each question, use the strongest source available:

| Question | Primary evidence | Why |
| --- | --- | --- |
| Rendered size, spacing, color, font as painted | Live computed styles and runtime measurement | Pixels are the truth for what is rendered |
| Intent, routes, state branches, conditional UI | Source code | Explains what can exist and when |
| What a specific moment looked like | Screenshot | Valid only for its captured instant |

Take the **union** of evidence when inventorying surfaces: source reveals branches, the live app reveals runtime states, screenshots reveal visual details. A screenshot-derived number is an **estimate**: label it with confidence and never treat it as measured fact. If evidence is absent, mark the value `unknown`; do not invent it.

Normalize the observation frame before any comparison (see `references/reference-spec.md`). Reference and candidate must agree on viewport in CSS px, device pixel ratio, zoom, scrollbar behavior, theme, locale, loaded fonts, and frozen time/data/animations. A comparison across mismatched frames is invalid even if it looks close.

## Artifacts

Write everything to `.agents/ui-replication/` in the consuming project. The templates in `references/` are generic starting points; the files in the project are generated evidence and must not be recycled across tasks.

```
.agents/ui-replication/
  frame.yaml
  coverage.yaml
  tokens.yaml
  components.yaml
  states.yaml
  target-layout/<surface>/<state>@<viewport>.yaml
  actual-layout/<surface>/<state>@<viewport>.yaml
  diff/ledger.yaml
  diff/gate-report.md
  screenshots/
  uncertainties.md
```

The layout schema, token/component schema, coverage schema, and ledger schema are in `references/reference-spec.md`. Read it before transcribing the first surface. The gate definitions, extraction procedure, and diff mechanics are in `references/verification.md`. Read it before comparing. The final screenshot protocol is in `references/screenshot-gate.md`; read it only when code gates are clean enough to capture. Framework-native parity tests live in the consuming repo's normal test tree, not under `.agents/`; record their paths and results in `diff/gate-report.md`.

## Workflow

### 0. Frame, evidence, coverage

Fill `frame.yaml` first. Pin viewport, DPR, zoom, theme, locale, font readiness, scrollbar mode, and a stable data/time state. Capture the reference in that frame and use the same frame for the candidate.

Build `coverage.yaml` as a preflight inventory of **every surface × state × viewport** before writing UI code:

- Routes and nested routes.
- Settings panels, tabs, sub-panels, and nested dialogs; a dialog opened from a dialog is its own surface.
- Modals, drawers, popovers, menus, toasts, tooltips, command palettes.
- Responsive versions: at minimum the reference's breakpoints plus a narrow and a wide viewport.
- States: default, hover, focus, active, disabled, loading, error, empty, overflow, long content, and any state the source exposes.

Each row needs an entry path or trigger, the evidence reference, and a status: `planned` (inventoried, not yet verified), `covered` (gates passed), `open` (in scope but unverified or failing), `unknown` (evidence was sought but is unobserved or unresolved), `deviated`, or `out-of-scope`. `unknown` is a first-class blocking status, not a synonym for a bounded estimate: the row is not verified and cannot pass. `deviated` and `out-of-scope` require explicit user/product approval recorded on the row; an agent cannot self-declare its own gaps away. `unknown`, `planned`, and `open` rows block parity and completion unless a narrow approved deviation covers that exact item. No silent omissions. If source and live app disagree about a surface's existence, record both and prefer live evidence for what renders.

A pre-build inventory may come from source branches and reference material even when a surface is not yet directly observed; that is not evidence that it renders correctly. A state or surface only becomes `covered` after runtime/state evidence has been captured and its gates pass. Merely listing a row `planned` is not scope reduction, and listing it `covered` before verification is not evidence.

### 1. Target spec (A)

Transcribe the reference into the layout schema **before implementing**, using the reference only. This is Spec A. Do not backfill from the candidate later, and do not let implementation plans change the transcription.

- Use stable semantic keys, not array indices.
- Record parent, sibling order, role, flow/layout relationships, sizing, padding/margin/border, overflow, stacking, bounds, real text, assets, and component variants/states.
- Flow relationships are primary; x/y/width/height are derived observations for checking the flow. Do not satisfy parity by absolutely positioning everything.
- Cap repetitive leaves (for example, a list of 40 identical rows) by recording representative items and the count/pattern, but never collapse a meaningful distinct component.
- Mark screenshot-estimated values with `confidence: estimate`; mark unknowns `unknown`.

Also record design tokens and component-level computed styles (typography, color, borders, radii, shadows, spacing, opacity, transitions) in `tokens.yaml`, `components.yaml`, and `states.yaml`. Group raw values into roles, but keep the resolved values that will actually be compared.

### 2. Build

Implement against Spec A. Reuse the reference's structure and design tokens where the project allows. Keep the implementation's source readable: it is evidence of intent, not proof of output.

Strongly prefer executable, language/framework-native parity tests when the consuming stack supports them, using the repo's existing test tooling. They are a deterministic projection of Spec A into the ordered gates, not a replacement for the neutral specs; read `references/executable-parity.md` before writing them. Do not add dependencies or a package manifest for this skill.

If you discover a surface or state that the inventory missed, add it to `coverage.yaml` with evidence before building it. That is a coverage correction, not a reason to quietly expand scope.

### 3. Actual spec (B)

After rendering, extract the candidate **mechanically wherever possible**: DOM/accessibility/native inspection, geometry APIs, and computed styles. Reading implementation source alone verifies intent, not output. If an agent must transcribe visually, it needs fresh eyes and must not consult the implementation's intent while doing so; mark those values as estimates.

Use the identical schema, node keys, units, property set, and node granularity as Spec A. A diff between different granularities is meaningless. Normalize both sides with the same canonicalization contract before diffing (numbers, color representation, enum names, box arrays, null handling). Assert that extraction is non-empty and report the emitted/raw coverage count: every covered surface/state/viewport must yield a spec with real nodes, or the gate fails rather than producing a false clean diff.

### 4. Diff and the ordered code gates

Compare A and B in this order. Do not polish a later gate while an earlier one fails.

| Gate | Checks | Blocks on failure |
| --- | --- | --- |
| 0 | Frame match, evidence provenance, coverage completeness | Everything |
| 1 | Surface parity: same surfaces, entry paths, states, viewports | Gates 2–7 |
| 2 | Hierarchy, parent/child, sibling order, roles, content existence | Gates 3–7 |
| 3 | Geometry and box model: flow, direction, alignment, gap, sizing, bounds, padding/margin, border width | Gates 4–7 |
| 4 | Content, assets, icons, truncation/wrapping, component variants | Gates 5–7 |
| 5 | Visual style and tokens: resolved typography, color, border style/color, radius, shadow, opacity | Gates 6–7 |
| 6 | States, interactions, and behavior: hover/focus/disabled/loading/error/empty, nested dialogs, overflow, navigation, validation, persistence, keyboard paths | Gate 7 |
| 7 | Final computer-use screenshot comparison | Completion |

Structural mismatches block token polish and screenshot capture. Verify the largest/root cause first: a wrong parent or flow direction often explains a dozen coordinate errors. After every fix, regenerate the affected Actual artifacts and re-run the diff; never mark an item fixed against stale extraction.

Executable framework tests, when the stack supports them, are graded as projections of these same expectations and tolerances. A required test that is failing or unexecuted blocks its owning gate and gate 7. A green suite does not replace the neutral diff or the final screenshot gate.

Write every discrepancy to `diff/ledger.yaml` with a stable surface/node ID, category, expected vs actual, delta and tolerance, evidence paths, and status (`open`, `fixed`, `accepted`, `unknown`). Fixed requires re-extraction and a clean re-diff. Accepted requires explicit user or product approval recorded in the ledger. Never relax a tolerance silently or wave through "close enough".

Default tolerances, declared before comparison: structure, hierarchy, order, and content existence are exact when visually observable; fixed geometry ±1 CSS px; flow-dependent geometry and type ±2 CSS px; text content exact when legible; resolved colors exact; shadows and transitions compared when observed. A target value read from a screenshot is a bounded estimate with a declared numeric uncertainty: a delta inside the band is not a proven mismatch and a delta outside it is a defect. If the evidence cannot support a defensible range, the value is `unknown`, the owning gate cannot pass, and it does not proceed to gate 7 for resolution. Tighter user requirements win. Passing against an estimated target is estimate-limited code parity, not proof that the estimate was measured.

### 5. Final computer-use screenshot gate

Only after gates 0–6 pass for a surface/state, use the available computer-use, GUI, or browser automation for gate 7. Declare one of two target modes:

- **A — live reference (preferred):** drive **both** reference and candidate through equivalent routes and states at the same viewport, DPR, theme, data, and font-ready condition.
- **B — static reference captures:** drive the candidate through the corresponding route/state and compare against user-supplied reference capture(s) at their original dimensions, crop, and content origin. Record what is known/unknown about the capture's frame and state; never stretch or reframe to force a match. This proves parity only for what the captures show.

In both modes:

- Capture the full surface plus focused regions (header, dialog, form row, footer).
- Compare same-dimension images; use overlays, side-by-side, or perceptual/pixel diffs when available.
- Record masks and dynamic regions explicitly.
- Inspect every residual difference at full size, not just a score, and route it back to gate 2, 3, 4, 5, or 6 and re-run.
- Require a meaningful, non-blank, aligned capture. A percentage alone is not a verdict.
- Account for anti-aliasing and rendering differences without using them to hide real errors.

If computer-use or candidate access is impossible, report the visual gate as **blocked** and do not claim completion. A missing live reference blocks only when there are no usable supplied captures for that row; otherwise use mode B. Routes/states/viewports that no capture shows stay `unknown`/`open` (or approved out-of-scope), never invented. A blocked run may be a complete, honest report, but it is never a completed replication. The gate is tool-agnostic; use whatever automation is available rather than assuming one vendor.

Only a bounded estimate may proceed to gate 7 for adjudication. An unbounded/`unknown` required value leaves its owning code gate incomplete: gate 7 is not run as a parity gate for that row, and additional evidence gathering belongs to gate 0 / target evidence, not gate 7. If gate 7 finds a defect on an eligible row, route it back to the owning gate, fix it, re-extract, and rerun gate 7.

Behavior is verified inside gate 6, before screenshot capture. In mode A, exercise navigation, validation, persistence, keyboard paths, and nested dialog flows on both apps and compare. In mode B, drive the candidate through its own required route to each supplied static state and compare only the behavior/state transitions documented by captured states, source, or user evidence. Static captures cannot prove unobserved reference behavior: report behavioral parity as **not established** and leave those rows `unknown`/`open`. If the user's task is scoped to supplied static UIs, gate 6 may pass the evidenced state-coverage requirement without claiming unobserved behavioral parity; that outcome is **"static visual replication complete for supplied captures"**, not a full interactive clone. If full behavioral replication was requested and target behavior evidence is missing, the broader completion remains blocked. Do not self-waive a requested behavioral requirement.

Record results in `diff/gate-report.md`. Gate 7 runs only after all evidenced in-scope UI state routes are exercised and stable. If a later behavior or rendering change alters a captured surface/state, that gate-7 result is invalid and must be rerun; any code change reruns the affected earlier gates and then gate 7.

## Definition of done

Code parity means gates 0–6 pass on extracted Actual artifacts. Visual parity means gate 7 passes on matched captures with no unexplained residual. Completion requires both, limited to rows with usable target evidence. When the target is a static capture set and behavior was not observed, the honest completion label is **static visual replication complete for supplied captures**, not full interactive clone completion; full behavioral completion requires evidenced behavior. A blocked or unknown gate, an unverified row, or a run that merely ended is not completion.

The final report lists:

- Gate results per surface/state/viewport.
- Screenshot gate result per surface/state, including target mode (A live / B static), masks, residual classes, and the limits of static evidence.
- Every accepted deviation with its approval, and every remaining unknown/uncertainty.
- Evidence paths for target specs, actual specs, ledger, captures, and report.

Do not claim "exact" or "pixel-perfect" beyond what the evidence supports. State the frame, tolerances, and residual uncertainty. A blocked visual gate is a truthful run outcome and still not a completed replication; an unverified success claim is not either.
