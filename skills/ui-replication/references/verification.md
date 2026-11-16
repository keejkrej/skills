# Verification: extraction, code gates, and diff discipline

Read this after `references/reference-spec.md` and before the first comparison. Screenshots are not part of this file; they are gate 7 in `references/screenshot-gate.md` and run only after code parity.

## Contents

- [Code parity vs visual parity](#code-parity-vs-visual-parity)
- [Gate 0: frame, evidence, coverage](#gate-0-frame-evidence-coverage)
- [Extracting the actual spec](#extracting-the-actual-spec)
- [Gates 1–6](#gates-16)
- [Executable framework tests](#executable-framework-tests)
- [Normalization before comparison](#normalization-before-comparison)
- [Field-level diff](#field-level-diff)
- [Tolerances and estimates](#tolerances-and-estimates)
- [Fix loop and re-extraction](#fix-loop-and-re-extraction)
- [Done criteria](#done-criteria)

## Code parity vs visual parity

**Code parity** means gates 0–6 pass between Spec A (reference transcription, captured before implementation) and Spec B (extracted from the rendered candidate), field by field, on the same schema after both sides pass through the same canonicalization contract. A diff of differently shaped or differently normalized artifacts is not a comparison. Spec A may be measured (live reference) or a bounded screenshot estimate. Code parity against an estimated target is parity against the available reference evidence with declared estimate limits; it is never proof that the estimated values were measured facts.

**Estimate-vs-measured rule.** Keep reference facts and candidate measurements labeled. Structural and content fields that are visually observable — hierarchy, parent, order, roles, text, assets, content existence — can still be exact in a screenshot-only case. Numeric geometry, spacing, type, and style fields compared against an estimate use the declared estimate uncertainty band for that field class. A delta inside the band is not a proven mismatch; a delta outside it is a defect. A bounded estimate must have a defensible declared numeric range; if the evidence cannot support such a range, the field is `unknown` and the owning code gate cannot pass. Do not use gate 7 as a catch-all to resolve an unbounded unknown.

**Unknown vs bounded estimate.** `unknown` means the field is unresolved and has no defensible value; it blocks parity and completion. A bounded estimate has a value and a declared uncertainty; it can support estimate-limited code parity and does not require a manual approved deviation for every estimated field. Do not convert a blocking unknown into an estimate to make a gate pass.

**Bounded estimate vs unbounded unknown at gate 7.** Only a bounded estimate may proceed to gate 7 for adjudication. If a required value is unbounded/unknown, its owning code gate is incomplete and gate 7 is not run as a parity gate for that row and cannot make it complete. Collecting more captures does not turn the unknown into a pass, and gate 7 is not a resolution pass. If additional evidence is needed to bound the unknown, gather it under gate 0 / target evidence, not gate 7. If gate 7 discovers a real defect on an otherwise eligible row, route it back to the owning gate, fix, re-extract, and rerun gate 7.

**Visual parity** means gate 7 passes on aligned candidate captures against the target evidence — a live reference in mode A or user-supplied static captures in mode B — with every residual difference explained or routed back to a code gate.

They are separate claims. A clean code diff does not prove the screenshot matches, because extraction can miss paint order, image content, font rasterization, or effects. A matching screenshot does not prove code parity, because the screenshot covers one instant and can hide unvisited surfaces, states, and responsive branches. Completion requires both.

## Gate 0: frame, evidence, coverage

Fail gate 0 and stop; nothing downstream is meaningful.

1. `frame.yaml` exists and the reference and candidate values are equal for viewport, DPR, zoom, scrollbar mode, theme, locale, direction, font status, motion, time, and data.
2. Every coverage row has a real evidence reference (capture path, source path, or runtime note) and a status. Only `covered` counts toward completion; `planned`, `open`, and `unknown` block until verified, and `deviated` / `out-of-scope` need an approval note and evidence reference. An agent cannot move its own failing or unverified rows into `deviated` / `out-of-scope`.
3. The union of source routes/branches and live/runtime surfaces is represented. Cross-check each source route and each modal/drawer trigger against the inventory. A pre-build inventory may be based on source branches and references even before direct observation; it is a plan, not proof that the surface renders or passes.
4. Target layout files were generated before implementation. Check timestamps and the trace, not memory. A target file that contains candidate-specific keys or values is backfilled and invalid.
5. Every in-scope coverage row has a target spec with at least one node.

Useful checks:

```sh
# Every coverage id has a target file (adjust the path convention to the project).
awk '/^  - id:/ {print $3}' .agents/ui-replication/coverage.yaml

# A spec is non-empty. A count of 0 is a failed gate, not a clean diff.
grep -c '^  - key:' .agents/ui-replication/target-layout/app.settings.account/default@1280x800.yaml
```

If the count is zero or the file is missing, extraction failed. Do not write a "no differences" report.

## Extracting the actual spec

Prefer mechanical extraction from the running candidate:

- **Web:** DOM traversal plus `getBoundingClientRect()` and `getComputedStyle()`. Use the accessibility tree for roles and names.
- **Native desktop:** the platform accessibility tree (AX, UIA, AT-SPI) for roles/hierarchy/names, plus window/screen geometry APIs for bounds and the native style or theme inspector for colors and typography.
- **Hybrid apps:** the webview DOM for content and the host accessibility tree for chrome.

Start from the candidate's own structure; do not generate keys by copying the target spec. Reuse the same key convention because the schema requires comparable keys, but derive each node from what actually renders. If a target node is absent, that absence must appear as a diff, not be skipped. Reading implementation source remains insufficient even when it looks authoritative.

Extraction is a two-stage pipeline. First walk the raw rendered tree and capture measurements for every node. Then apply a deterministic, declared filter/collapse step that keeps meaningful components and drops empty or decorative wrappers. The same retention and collapse rules must run on both sides; raw walking may visit every node, but the emitted spec must stay within the bounded schema and must not lose a meaningful component, asset, or repeated row.

Illustrative pseudocode — **not runnable** and not a schema producer by itself. It shows the shape of a measurement adapter, not a finished extractor:

```
walk raw rendered tree from the surface root
  rect   = element.getBoundingClientRect()  # or native equivalent
  style  = computedStyle(element)           # or native theme inspector
  record {element, rect, style, children, text, assets}

filter/collapse with the declared rule
  drop empty wrappers and purely decorative nodes
  keep semantic roles, text, assets, and interactive nodes
  collapse repeated identical leaves to representatives + repeat count

map to schema
  key      = explicitMap(element) or captured semantic key
  bounds   = rootRelative(rect, rootRect, rootScroll)
  box      = numericTRBL(style)
  flow     = canonicalFlow(style)
  text     = exactOwnText(element) + nested-text rule
  style    = canonicalStyle(style)          # gate 5 fields only
```

**Field-normalization contract.** Resolve the measurement adapter to the canonical schema on both sides *before* any diff. A clean diff is meaningless if the two adapters emit different shapes. The adapter must be tested against a known fixture with known expected fields before its output is trusted.

- **Origin:** emit bounds relative to the surface root, not the viewport. Subtract the root's rect origin and account for the root's scroll offset. A dialog opened over a page uses the dialog surface's root, not the page origin.
- **Numbers:** emit CSS px as numbers, not shorthand strings. `12px` becomes `12`; `0.5px` stays `0.5`. Do not keep `cs.padding` as `"8px 10px"`.
- **Box model:** `padding`, `margin`, `border` are arrays in CSS shorthand order `[top, right, bottom, left]`. `border` is width only; width is a gate-3 dimension, while border style/color belong to gate 5. `radius` is not part of the layout box; record it in the component/token style where gate 5 reads it.
- **Gap:** a single number when row and column gaps are equal, otherwise `[row, column]`. `normal` and `0px` normalize to `0`.
- **Enums:** map computed values to the schema enums explicitly. `flex-start` → `start`, `space-between` → `between`, `flex-end` → `end`, `center` → `center`, `stretch` → `stretch`. Unmapped values become `unknown`, not raw CSS.
- **Sizing:** `size.w`/`size.h` express declared/inferred behavior (`fill`, `hug`, or a numeric fixed constraint) and are diffed at the size path. `bounds.w/h` are observed root-relative numeric geometry and are diffed at the bounds path. For fixed numeric sizes the two must agree within tolerance; do not replace one field with the other or emit contradictory values.
- **Keys:** map reference nodes to keys through an explicit capture/mapping adapter (data attributes, accessibility name/role, route/region map). Never rely on blind DOM child-index equality; indices change with wrappers and conditional nodes. Unmapped rendered nodes get a candidate-derived semantic key and surface as extras.
- **Text:** preserve exact text, including meaningful nested text. Aggregate a parent's text only by a declared rule that does not duplicate text already emitted on its children. Do not truncate before diffing exact content; truncate only display for review and keep the full value in the artifact.
- **Collapse:** apply the same declared repetition rule on both sides. If the target collapses 40 identical rows to 3 representatives + count, the actual must use the same rule and count, or the collapse itself becomes a diff.
- **Null/none:** canonicalize `none`, `normal`, `auto`, and empty strings per field. Do not let one side emit `null` and the other `"none"` for the same absence.
- **Output:** write the non-empty spec plus a coverage count (emitted nodes vs raw meaningful nodes, and collapsed count). A zero-node or near-empty file fails the gate; it is not a clean diff.

Measured vs measured is the strongest comparison; measured vs bounded estimate is allowed under the estimate rule above and must be reported as estimate-limited. Spec B must always be measured or mechanically extracted from the rendered candidate, never predicted from source. An estimate without a defensible numeric range is `unknown`, not bounded; it blocks code parity and is not sent to gate 7 for resolution. Gate 7 may falsify a bounded agreement but cannot establish measured numeric facts.

For a screenshot-only reference, transcription is unavoidable. Transcribe before looking at candidate code or screenshots of the candidate, mark `confidence: estimate`, declare estimate uncertainty per field class in the ledger, and keep measured candidate values for the actual side. Do not let an estimate masquerade as measured fact, and do not hide a mismatch that exceeds the declared estimate band.

Assert non-empty extraction for every covered surface/state before diffing:

```sh
for f in .agents/ui-replication/actual-layout/*/*.yaml; do
  n=$(grep -c '^  - key:' "$f")
  [ "$n" -gt 0 ] || echo "EMPTY: $f"
done
```

## Gates 1–6

Run in order. A failure blocks later gates for the affected surface/state, not necessarily for unrelated surfaces.

### Gate 1 — surface parity

- Every `covered` coverage id has both a target and an actual spec at each listed viewport.
- Entry path/trigger and state list match between target and actual.
- No extra surface in the candidate that the inventory did not account for; add it to coverage or record it as an intentional deviation.

### Gate 2 — hierarchy, parent, sibling order, content existence

- Node sets match: missing nodes, extra nodes, wrong parent, wrong role, wrong sibling order.
- Content existence: every text, asset, icon, and component node in the target exists in the actual (or the ledger records its absence).
- Compare by stable key first, then verify the tree shape. A correct set of nodes in the wrong hierarchy still fails.

Structural mismatches here block gate 3 and everything after. Fix parent/order/role before nudging pixels.

### Gate 3 — geometry and box model

- Flow type, direction, justify, align, wrap, and gap.
- Sizing (`fill`/`hug` or a numeric fixed dimension), padding, margin, border width, overflow, stacking. Spacing and size dimensions pass/fail here; token naming may be reported but is not re-gated.
- Bounds within tolerance, with attention to deltas that indicate a wrong parent or flow rather than a local offset.
- A repeated 8px offset on many siblings usually means one gap/padding error upstream. Fix the cause, not each leaf.

### Gate 4 — content, assets, icons, variants

- Exact text, wrapping, truncation, pluralization, and long-content behavior.
- Real asset identity and intrinsic size; icon variant and optical size.
- Component variant and size (`primary`/`danger`, `sm`/`md`) and any node-level state that changes content.

### Gate 5 — visual style and tokens

- Resolved typography: family, size, line height, weight, letter spacing.
- Color and background; border style and color; radius; shadows; opacity; transitions when observed. Border width is gate 3; border style/color and radius are gate 5. Radius lives in the component/token style, not the layout box.
- Compare resolved values as well as token roles. Two components using the same token name but resolving differently is a real defect.
- Group patterns into tokens after measuring, not before. Do not rename a raw value to a token to hide a mismatch.

### Gate 6 — states, interactions, and behavior

Drive each in-scope state and re-extract: hover, focus, active, disabled, loading, error, empty, overflow, long content, and every nested dialog/drawer/popover. Also exercise behavior that changes UI state — navigation, validation, persistence, keyboard paths, and nested dialog flows — and compare it against the reference when live evidence exists. Re-run gates 2–5 on the state's actual spec. A state that cannot be produced stays `open`/`unknown` (or `deviated` only with explicit approval), never assumed correct.

With static target captures (gate 7 mode B), drive the candidate through its own required route to each supplied state and compare only transitions documented by the captures, source, or user evidence. Behavioral parity not evidenced by those sources is **not established**, not a pass. If the task is scoped to supplied static UIs, gate 6 may pass the evidenced state-coverage requirement for those rows; the result is static visual replication, not a full interactive clone. If full behavioral replication is requested and target behavior evidence is missing, those rows stay `unknown`/`open` and block the broader completion.

Gate 6 must cover every in-scope UI state route before gate 7. Gate 7 is invalidated if later behavior or rendering work changes a captured surface/state; any such change reruns the affected earlier gates and then gate 7.

## Executable framework tests

When the consuming stack supports it, project Spec A into the repository's existing language/framework test suite. Read `references/executable-parity.md` for stack choices and derivation rules. The neutral specs remain canonical; tests are an executable projection with the same canonicalization and tolerance config.

- Derive expected values from independently captured Spec A/reference evidence, never from candidate constants or an auto-updated snapshot.
- Read actual values from the rendered runtime, not implementation values.
- Map semantic keys explicitly to stable test IDs/accessibility roles and assert nonempty selection/counts.
- Cover surfaces and structure first, then geometry/style/state/interactions/responsive rows.
- Match estimate confidence: exact for observable structure/content, estimate-bounded or `unknown` for uncertain numeric values. Do not assert unsupported layout behavior in DOM-only environments.
- Put tests in the repo's normal test location and record paths, commands, environment/viewports, counts, failures, timestamp/revision in `diff/gate-report.md`.
- Rerun affected tests and re-extract after every fix. A required failing or unexecuted test blocks its owning code gate and gate 7. A green suite does not replace the neutral diff or gate 7.

## Normalization before comparison

Normalize both sides with the same declared rules before diffing. This is separate from the artifact schema; it is how values are compared.

- **Numbers:** convert every CSS length to a number in CSS px (`16px` → `16`, `1rem` → the resolved px value from the frame). Compare numbers after normalization with the declared tolerance, never string shorthand.
- **Colors:** canonicalize to a single documented representation, for example `rgba(r, g, b, a)` with alpha in `[0,1]` (or a canonical hex with alpha). Do not compare `#5b8def` directly to `rgb(91, 141, 239)`; resolve tokens to the same representation first. Alpha is part of equality.
- **Borders and shadows:** expand shorthands into structured values: border `{width, style, color}` per side and shadow `{x, y, blur, spread, color, inset}` (one object or declared list). Compare the structured components, not the raw shorthand string.
- **Typography:** compare family, size, line height, weight, letter spacing, and style as separate numeric/string fields. Serialize `font-family` consistently (declared stack order preserved). Keep two distinct pieces of evidence: the declared stack and the actually loaded/resolved face. A matching declared stack does not prove the same resolved face; record font readiness and loaded families from the frame.
- **Null and absence:** canonicalize `none`, `normal`, `auto`, empty, and missing to the same documented sentinel per field. One side emitting `null` and the other `"none"` is a normalization bug, not a visual defect.
- **Exact vs tolerance:** compare geometry numbers with the tolerance table; compare resolved colors, border style, shadow structure, and text exactly after canonicalization.

Unnormalized comparisons are invalid even when they happen to look close.

## Field-level diff

Diff keyed by node key and property path. Compare in both directions:

- **Target → actual:** every expected node and property is present and equal.
- **Actual → target:** every rendered node and property is accounted for. Extra nodes are defects unless the ledger explains them.

Record one ledger row per discrepancy, with the smallest property path that explains it:

```yaml
- id: D-012
  node: settings.account.form.email
  gate: 5
  category: style
  expected: {font: {size: 14, lineHeight: 20, weight: 400}}
  actual: {font: {size: 14, lineHeight: 24, weight: 400}}
  delta: {px: 4, property: lineHeight}
  tolerance_px: 2
  severity: minor
  status: open
```

Do not average mismatches into a score. A single wrong parent or missing dialog is more important than twenty 1px font differences, and the ledger must show that priority through gate order and severity.

## Tolerances and estimates

Declare tolerances in `diff/ledger.yaml` before comparing:

| Category | Default | Notes |
| --- | --- | --- |
| Structure, hierarchy, order, roles, content existence | exact when visually observable | Screenshot-only references can still be exact here; unreadable/ambiguous structure is `unknown` |
| Text content and wrapping | exact when legible | Long-content behavior included; illegible text is `unknown`, not guessed |
| Fixed geometry (explicit width/height, fixed gaps) | ±1 CSS px | Rendering can round subpixel values |
| Flow-dependent geometry, bounds, typography size/leading | ±2 CSS px | Only when the flow relationship is correct |
| Comparison against a screenshot estimate | ± declared estimate uncertainty | Declare per field class in the ledger; inside band is not a proven mismatch, outside band is a defect |
| Colors | exact resolved value | Canonicalized first; alpha included |
| Border style/color, radii | exact after normalization | Border width follows geometry tolerance |
| Shadows | structured compare; exact when observed | Estimated shadow values stay uncertain |
| State presence | exact | An unvisited state is `open`/`unknown`, not passing |

A tighter user requirement overrides these defaults. Record the override in the ledger.

Screenshot estimates are bounded observations, not measurements. Compare the measured actual against the estimate using the declared estimate uncertainty. A delta inside the band is not a proven mismatch; report it as estimate-limited code parity, not measured equality. A delta outside the band is a defect. If the evidence cannot support a defensible numeric range, the field is `unknown`, the owning code gate cannot pass, and gate 7 cannot resolve it; do not silently pass, silently fail, or treat the same uncertain capture as independent proof.

## Fix loop and re-extraction

1. Sort open defects by gate, then severity. Fix gate 2 before gate 3 before gate 5.
2. Fix the largest/root cause. Prefer one upstream fix over many leaf patches.
3. Regenerate the affected actual spec from the running candidate.
4. Re-diff the affected fields and update the ledger.
5. Mark `fixed` only when the re-extracted value passes. If the fix introduced a new defect, add it.
6. Do not re-run the screenshot gate on a surface whose code gate just regressed. Any code change invalidates prior captures for the affected surface/state; rerun the affected earlier gates, then gate 7.

Stale actual artifacts are a common false pass. If the implementation changed, the actual spec is invalid until re-extracted.

## Done criteria

For every in-scope coverage row:

- Gate 0 passes: frame matched, evidence present, coverage complete, target captured before implementation.
- Gates 1–6 pass on normalized artifacts, including gate 6 behavior. Spec B is measured/extracted; Spec A may be measured or a bounded estimate. Required executable framework checks that the run authored because the stack supports them must be executed and passing for done. A failing or unrun required check blocks its owning gate and gate 7/completion; an approved deviation may record an exact known mismatch separately but cannot turn a red/skipped test into a passing test or supply missing evidence. Optional checks that were not required may be listed as skipped limitations. Estimate-limited parity is reported as such, with the bands and limits listed. Each remaining failure must be a narrow `accepted` deviation approved by the user/product for that exact item, with an approval note and evidence reference. Blocking `unknown` fields are not estimates and are not waved through.
- Gate 7 passes per `references/screenshot-gate.md` on matched captures; it remains mandatory even when code parity is estimate-limited. A bounded estimate may proceed to gate 7; a blocking `unknown` keeps its owning gate incomplete and cannot be turned into a gate-7 pass. A blocked or unknown gate 7 is an honest run/report state but not completion.
- No in-scope row is left `planned`, `open`, or `unknown`, and no missing in-scope row may be relabeled `covered`, `deviated`, or `out-of-scope` by the agent to narrow scope.
- The gate report lists per-surface results and the evidence paths.

If any of these is missing, the correct status is incomplete. A run ending, a report existing, or a blocked gate is not a pass. Never convert an unrun or blocked gate into a pass, and never let aggregate wording turn unknown rows into done. When Spec A is estimated, state "code parity against available reference evidence (estimate-limited)" with the bounds and uncertainty — never "the target values were measured" or unqualified "exact".
