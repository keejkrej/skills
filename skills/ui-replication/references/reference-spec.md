# Reference spec and artifact schemas

Templates and schema for the artifacts in `.agents/ui-replication/`. These are deliberately small: a surface with 15–40 meaningful nodes is normal; a 900-node transcription of every DOM leaf is not usable. Cap repetitive leaves and keep every component that a reviewer would notice.

All files are framework-independent YAML unless stated otherwise. Use the same schema version, node keys, units, and property set for target (A) and actual (B).

## Contents

- [Frame](#frame)
- [Coverage inventory](#coverage-inventory)
- [Layout spec](#layout-spec)
- [Design tokens and components](#design-tokens-and-components)
- [States](#states)
- [Diff ledger](#diff-ledger)
- [Gate report](#gate-report)
- [Uncertainties](#uncertainties)
- [Minimal worked example](#minimal-worked-example)

## Frame

`frame.yaml` pins the observation conditions. Both reference and candidate must match every equality field before comparison.

```yaml
schema: ui-frame/1
viewport: {w: 1280, h: 800}     # CSS px
dpr: 2
zoom: 1
scrollbars: overlay            # overlay | classic | hidden
theme: dark                    # resolved, not "system"
locale: en-US
direction: ltr
fonts:
  status: ready                # ready | fallback | unknown
  families: [Inter, "IBM Plex Mono"]
  source: bundled | system | remote
motion: disabled               # disabled for comparison
time: "2026-01-15T10:00:00Z"   # frozen if the UI shows time
data: seeded                   # seeded | empty | long-content | fixture path
browser: "Chromium 131, macOS" # informational only
```

Record any unavoidable mismatch in `uncertainties.md`; do not compare across it.

## Coverage inventory

`coverage.yaml` is the preflight contract. A surface is any distinct visual region reachable by an entry path. A dialog opened from a dialog is a separate surface. States are rows, not footnotes.

```yaml
schema: ui-coverage/1
surfaces:
  - id: app.settings.account
    kind: route                 # route | panel | tab | modal | drawer | popover | menu | toast | tooltip
    entry: "Settings > Account"
    viewports: ["1280x800", "390x844"]
    states: [default, hover, focus, disabled, loading, error, empty, overflow, long-content]
    evidence:
      - screenshots/ref/settings-account-default.png
      - src/routes/settings/account.tsx
    status: covered             # planned | covered | open | unknown | deviated | out-of-scope
    approval: null              # required for deviated / out-of-scope
    reason: null

  - id: app.settings.account.delete-dialog
    kind: modal
    entry: "Settings > Account > Delete account"
    parent: app.settings.account
    viewports: ["1280x800", "390x844"]
    states: [default, confirming, error]
    evidence: [screenshots/ref/delete-dialog.png]
    status: covered
    approval: null
    reason: null

  - id: app.settings.account.delete-dialog.confirm-nested
    kind: modal
    entry: "Delete account > confirm typed name > Delete"
    parent: app.settings.account.delete-dialog
    viewports: ["1280x800"]
    states: [default, disabled]
    evidence: [screenshots/ref/delete-confirm-nested.png]
    status: out-of-scope
    approval: "user approved omission 2026-01-15 (thread ref: msg-42)"
    reason: "Requires production-only account state"
```

Rules:

- List every route and every nested panel/tab/dialog you can reach. Source is good for finding branches; the live app confirms them.
- `status` lifecycle: `planned` (inventoried, not verified), `covered` (gates passed), `open` (in scope but unverified or failing), `unknown` (evidence was pursued but is unobserved or unresolved), `deviated` / `out-of-scope` (narrow, approved exclusion). `unknown` is a real schema status and blocks parity; it is not a bounded estimate.
- Only `covered` counts as verified. `planned`, `open`, and `unknown` rows remain open and block parity and completion. An agent may not self-declare a failing or unverified row `deviated` or `out-of-scope`; those statuses require an explicit user/product approval note and evidence reference, and the approval must cover that exact item. No aggregate wording ("mostly covered") converts them to done.
- A state that cannot be produced (permissions, backend failure) is still listed with a reason and the status it honestly has; it is not silently dropped.
- `evidence` must point to a real capture, source path, or runtime note. "Seen once" is not evidence.
- A pre-build inventory can be assembled from source branches and reference material before direct observation; that makes rows `planned`, not `covered`. Runtime/state evidence is required before a row can pass.
- Never mark an in-scope missing row `covered`. Add missed surfaces first, with evidence, then build.
- Update this file when implementation reveals a missed surface. Add first, then build.

## Layout spec

`target-layout/<surface>/<state>@<viewport>.yaml` and the matching `actual-layout/...` file use this schema. Store bounds in CSS px, relative to the surface root.

```yaml
schema: ui-layout-spec/1
meta:
  surface: app.settings.account
  state: default
  viewport: {w: 1280, h: 800}
  dpr: 2
  source: target              # target | actual
  method: screenshot          # screenshot | dom | ax | native | hybrid
  confidence: estimate        # estimate | measured | mixed
  evidence: [screenshots/ref/settings-account-default.png]
  generated: "2026-01-15T10:05:00Z"
nodes:
  - key: settings.account
    parent: null
    order: 0
    role: main
    bounds: {x: 0, y: 0, w: 1280, h: 800}
    flow: {type: flex, dir: column, justify: start, align: stretch, wrap: nowrap, gap: 0}
    box: {padding: [0, 0, 0, 0], margin: [0, 0, 0, 0], border: [0, 0, 0, 0]}
    size: {w: fill, h: fill}
    overflow: hidden
    z: 0

  - key: settings.account.nav
    parent: settings.account
    order: 0
    role: navigation
    bounds: {x: 0, y: 0, w: 240, h: 800}
    flow: {type: flex, dir: column, justify: start, align: stretch, gap: 4}
    box: {padding: [16, 12, 16, 12], margin: [0, 0, 0, 0], border: [0, 0, 0, 0]}
    size: {w: 240, h: fill}
    overflow: auto
    z: 0

  - key: settings.account.nav.item.account
    parent: settings.account.nav
    order: 2
    role: link
    bounds: {x: 12, y: 96, w: 216, h: 36}
    flow: {type: flex, dir: row, justify: start, align: center, wrap: nowrap, gap: 8}
    box: {padding: [8, 10, 8, 10], margin: [0, 0, 0, 0], border: [0, 0, 0, 0]}
    size: {w: fill, h: 36}
    overflow: hidden
    z: 0
    variant: nav-item
    state: active
    text: {value: Account, wrap: nowrap, lines: 1, overflow: ellipsis}
    asset: null
```

Field rules:

- `key`: stable, semantic, dot-scoped. Derive from route + region + component, never from DOM order.
- `parent` / `order`: sibling order is an integer starting at 0. Parent plus order defines structure; bounds merely verify it.
- `role`: semantic role (`main`, `navigation`, `button`, `dialog`, `textbox`, `heading`, `list`, `listitem`, `img`, `separator`, `tooltip`, …). Prefer accessibility roles.
- `flow.type`: `flex | grid | block | inline | absolute | fixed | overlay`. For grid add `columns` and `rows` inside `flow`. For block add `spacing` if relevant.
- `flow`: `dir` (`row | column`), `justify` (`start | center | end | between | around | evenly`), `align` (`start | center | end | stretch | baseline`), `wrap`, `gap` (number or `[row, column]`).
- `box`: `padding`, `margin`, and `border` widths in CSS shorthand order `[top, right, bottom, left]`, as numbers. Border style and color are visual style (gate 5) and live in `components.yaml` / `states.yaml`; this layout box carries width only. `radius` is a visual style value, recorded in the component/token style, not in the layout box.
- `size`: declared or inferred sizing behavior. `w`/`h` are `fill | hug | <number>` in CSS px; use the numeric form when a fixed constraint is known. If the constraint is only observed, record `fixed` with `fixed_from: bounds` rather than inventing a conflicting number. `size` is diffed at its own path and must not contradict `bounds`.
- `bounds`: observed root-relative numeric x/y/w/h in CSS px (root rect origin and scroll offset applied), not viewport coordinates. Bounds are diffed at the bounds path as geometry; they are derived from flow and are compared with tolerance, not a license to position absolutely. For a fixed numeric size, verify `size` and `bounds` agree within tolerance; do not silently swap one field for the other.
- `overflow`: `visible | hidden | auto | scroll | clip`; add `scroll: {x, y}` when scrolled away from origin.
- `z`: stacking order within the parent. Record only when overlaps occur.
- `text`: exact string plus `wrap` (`wrap | nowrap | balance`), `lines`, `overflow` (`ellipsis | clip | visible`). Preserve real content, including long strings.
- `asset`: `{kind: icon | image | avatar | illustration, ref, alt, w, h}` or null. Use the real asset identity, not a placeholder name.
- `variant` / `state`: component variant and current state as rendered (`primary`, `ghost`, `danger`; `default`, `hover`, `focus`, `disabled`, `loading`, `error`).

Repetitive leaves: record 2–3 representative items, add `repeat: {count: 40, key_pattern: "row.{index}"}` on the parent, and keep distinct rows that differ meaningfully. Do not collapse a dialog, a toolbar, or anything with different behavior.

Unknown values: write `unknown` (or `null` where the schema permits) and add the reason to `uncertainties.md`. Do not estimate silently. Values read from a screenshot are bounded estimates: mark the file `confidence: estimate` (or `mixed`) and declare per-field-class uncertainty in the ledger. Structure, roles, order, and text can still be exact in that file when clearly observable; numeric geometry and style stay bounded. `unknown` is not a bounded estimate and still blocks parity. An `unknown` field keeps its owning gate incomplete; gate 7 is not run as a parity gate for that row and cannot pass it or convert it into completion. Additional evidence needed to bound or resolve the unknown belongs to gate 0 / target evidence, not gate 7. A bounded estimate may proceed to gate 7 for adjudication.

## Design tokens and components

`tokens.yaml` groups resolved values into roles. The role name is for humans; the resolved value is what gets diffed. Hex colors in these examples are human-readable notation: canonicalize both sides to one documented representation (for example `rgba(r, g, b, a)`) before comparing, never hex to `rgb(...)` directly.

```yaml
schema: ui-tokens/1
evidence: [screenshots/ref/settings-account-default.png, "live computed styles"]
color:
  bg.canvas: "#111318"
  bg.surface: "#191c23"
  bg.surface-hover: "#20242d"
  fg.primary: "#e8eaf0"
  fg.muted: "#9aa2b1"
  border.subtle: "#2a2f3a"
  accent.default: "#5b8def"
  danger.default: "#e5534b"
typography:
  text.body: {family: Inter, size: 14, lineHeight: 20, weight: 400, letterSpacing: 0}
  text.label: {family: Inter, size: 12, lineHeight: 16, weight: 500, letterSpacing: 0.2}
  text.title: {family: Inter, size: 20, lineHeight: 28, weight: 600, letterSpacing: -0.2}
spacing: {xs: 4, sm: 8, md: 12, lg: 16, xl: 24}
radius: {sm: 4, md: 6, lg: 10, full: 9999}
shadow:
  popover: {x: 0, y: 8, blur: 24, spread: 0, color: "rgba(0,0,0,.32)", inset: false}
motion:
  hover: "background-color 120ms ease-out"
```

`components.yaml` records the resolved computed style of each component variant. Compare values, not just token names; a component can resolve a token differently. Border values here are the gate-5 source for style and color; the layout spec carries border width only. Store border and shadow as structured values (`border: {width, style, color}`, `shadow: {x, y, blur, spread, color, inset}`), not raw CSS shorthand strings. A readable raw string may be kept as a separate note, but the normalized structure is what gets diffed.

```yaml
schema: ui-components/1
components:
  button.primary:
    node: settings.account.form.save
    variant: primary
    properties:
      background: "#5b8def"
      color: "#ffffff"
      border: {width: 1, style: solid, color: "#5b8def"}
      radius: 6
      padding: [8, 14, 8, 14]
      gap: 8
      font: {family: Inter, size: 14, lineHeight: 20, weight: 500}
      shadow: none
      opacity: 1
    exceptions:
      - scope: "settings.account.form.save[disabled]"
        background: "#2a2f3a"
        color: "#6b7280"

  dialog.surface:
    node: settings.account.delete-dialog.panel
    properties:
      background: "#191c23"
      border: {width: 1, style: solid, color: "#2a2f3a"}
      radius: 10
      padding: [20, 20, 20, 20]
      shadow: {x: 0, y: 16, blur: 48, spread: 0, color: "rgba(0,0,0,.4)", inset: false}
      width: 440
```

## States

`states.yaml` captures observed state deltas and records the evidence status for each state group. Only record values you actually observed; an unobservable state is represented with `status: unknown` plus a coverage reference (or listed in coverage with status `unknown` and a reason). A field may also be `unknown` inside an observed group when only part of the state was resolvable.

```yaml
schema: ui-states/1
states:
  button.primary:
    status: observed
    evidence: [screenshots/ref/settings-account-default.png, "live computed styles"]
    default: {background: "#5b8def", color: "#ffffff"}
    hover: {background: "#6f9bf2"}
    focus: {outline: {width: 2, style: solid, color: "#8bb2ff"}, outlineOffset: 2}
    active: {background: "#4a7ade"}
    disabled: {background: "#2a2f3a", color: "#6b7280", cursor: not-allowed}
    loading: {content: spinner, label: "Saving…", disabled: true}
  textbox.search:
    status: observed
    evidence: ["live computed styles"]
    default: {border: {width: 1, style: solid, color: "#2a2f3a"}}
    focus: {border: {width: 1, style: solid, color: "#5b8def"}, shadow: {x: 0, y: 0, blur: 0, spread: 3, color: "rgba(91,141,239,.25)", inset: false}}
    error: {border: {width: 1, style: solid, color: "#e5534b"}, message: "Enter a valid email"}
    empty: {placeholder: "Search settings"}
  textbox.billing:
    status: unknown
    coverage: app.settings.billing
    reason: "Requires a test payment failure; not reproducible"
```

Rules: every `unknown` state group must resolve to a coverage row with status `unknown` (or be that row's evidence), so `unknown` always has a schema location. Do not put an unknown value in `states.yaml` without either `status: unknown` on its group or a coverage reference.

## Diff ledger

`diff/ledger.yaml` is the durable record. No discrepancy may live only in a chat message.

```yaml
schema: ui-diff-ledger/1
run:
  started: "2026-01-15T10:00:00Z"
  frame: frame.yaml
tolerances:
  structure: exact
  content: exact
  geometry_fixed_px: 1
  geometry_flow_px: 2
  typography_px: 2
  color: exact-resolved
defects:
  - id: D-001
    surface: app.settings.account
    state: default
    viewport: "1280x800"
    node: settings.account.form.email
    gate: 3
    category: geometry          # hierarchy | order | content | asset | geometry | token | style | state | behavior | screenshot
    expected: {x: 272, y: 184, w: 400, h: 40}
    actual: {x: 272, y: 196, w: 400, h: 40}
    delta: {px: 12, axis: y}
    tolerance_px: 2
    severity: major             # blocker | major | minor
    evidence:
      - target-layout/app.settings.account/default@1280x800.yaml
      - actual-layout/app.settings.account/default@1280x800.yaml
    status: open                # open | fixed | accepted | unknown
    approval: null              # required when status=accepted
```

Status discipline:

- `fixed` only after the Actual artifact was regenerated and the field re-diffed clean.
- `accepted` only with a named approver and reason; never self-approve.
- `unknown` when the evidence cannot resolve it; carry it into the final report.
- Do not delete a defect to make the ledger look clean. Close it under one of the statuses above.

## Gate report

`diff/gate-report.md` is the human-readable summary produced from the ledger and captures.

```markdown
# UI replication gate report

Frame: 1280x800, DPR 2, dark, en-US, fonts ready, motion disabled
Tolerances: structure/content exact; fixed geometry ±1px; flow/type ±2px; colors exact

## Code parity (gates 0–6)

| Surface | State | Viewport | 0 | 1 | 2 | 3 | 4 | 5 | 6 | Result |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| app.settings.account | default | 1280x800 | pass | pass | pass | fail D-001 | pass | pass | pass | incomplete |
| app.settings.account.delete-dialog | default | 1280x800 | pass | pass | pass | pass | pass | pass | pass | code-clean |

## Visual parity (gate 7)

| Surface | State | Capture pair | Masks | Residual class | Result |
| --- | --- | --- | --- | --- | --- |
| app.settings.account | default | — | — | — | not run: blocked by gate 3 (D-001 open) |
| app.settings.account.delete-dialog | default | screenshots/cmp/delete-default-{ref,cand}.png | none | none | pass |

## Executable checks

| Gate | Test path | Command | Environment / viewports | Counts | Result |
| --- | --- | --- | --- | --- | --- |
| 2 | tests/ui/settings-dialog.structure.test.ts | `pnpm test -- settings-dialog` | Chromium, 1280x800 | 12 selected / 12 passed | pass |
| 3 | tests/ui/settings-dialog.geometry.test.ts | `pnpm test -- settings-dialog.geometry` | Chromium, 1280x800 | 8 selected / 8 passed | pass |

## Deviations and unknowns

- D-004 accepted by <approver>: drawer width 320 vs 324, platform scrollbar width.
- Font smoothing differs on Linux CI; no layout impact.

## Evidence

- coverage.yaml, frame.yaml
- target-layout/, actual-layout/, diff/ledger.yaml
- screenshots/ref/, screenshots/candidate/, screenshots/cmp/
```

## Uncertainties

`uncertainties.md` is a short list, not an essay. Each entry names what is unknown, why, and whether it blocks completion.

```markdown
# Uncertainties

- Settings > Billing error state: reference requires a test payment failure; not reproducible. Coverage status: `unknown`. Blocks the Billing surface and full completion until runtime/state evidence or a named approved deviation is recorded in coverage with an approval note and evidence reference.
- Dialog shadow read from screenshot at DPR 2; alpha 0.32 with declared uncertainty ±0.08. Estimate-limited: gate 5 passes only within that band, and gate 7 adjudicates the visible residual but cannot establish the measured fact. If alpha cannot be bounded, the value is `unknown`, gate 5 stays incomplete, and gate 7 is not run for that row.
```

## Minimal worked example

A compact target/actual pair for one dialog. The actual file uses the same keys and property set.

```yaml
schema: ui-layout-spec/1
meta:
  surface: app.settings.account.delete-dialog
  state: default
  viewport: {w: 1280, h: 800}
  dpr: 2
  source: target
  method: screenshot
  confidence: estimate
  evidence: [screenshots/ref/delete-dialog.png]
nodes:
  - key: delete-dialog
    parent: null
    order: 0
    role: dialog
    bounds: {x: 420, y: 260, w: 440, h: 280}
    flow: {type: flex, dir: column, justify: start, align: stretch, gap: 16}
    box: {padding: [20, 20, 20, 20], margin: [0, 0, 0, 0], border: [1, 1, 1, 1]}
    size: {w: 440, h: hug}
    overflow: visible
    z: 100
    variant: danger
    state: default

  - key: delete-dialog.title
    parent: delete-dialog
    order: 0
    role: heading
    bounds: {x: 441, y: 281, w: 398, h: 28}
    flow: {type: block}
    box: {padding: [0, 0, 0, 0], margin: [0, 0, 0, 0], border: [0, 0, 0, 0]}
    size: {w: fill, h: hug}
    overflow: visible
    z: 0
    text: {value: "Delete account?", wrap: nowrap, lines: 1, overflow: visible}

  - key: delete-dialog.actions
    parent: delete-dialog
    order: 2
    role: group
    bounds: {x: 441, y: 480, w: 398, h: 36}
    flow: {type: flex, dir: row, justify: end, align: center, gap: 8}
    box: {padding: [0, 0, 0, 0], margin: [0, 0, 0, 0], border: [0, 0, 0, 0]}
    size: {w: fill, h: hug}
    overflow: visible
    z: 0
```

A diff entry for this surface when the actual dialog is 8px narrower and actions are left-aligned:

```yaml
- id: D-007
  surface: app.settings.account.delete-dialog
  state: default
  viewport: "1280x800"
  node: delete-dialog
  gate: 3
  category: geometry
  expected: {w: 440, justify: end}
  actual: {w: 432, justify: start}
  delta: {px: 8}
  tolerance_px: 2
  severity: major
  status: open
```

The width and the alignment are separate root causes; fix the sizing constraint and the actions flow before touching shadows or colors.
