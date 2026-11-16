# Final computer-use screenshot gate

This is gate 7, the last gate. Run it only for a surface/state whose code gates 0–6 already pass. It does not replace code parity, and code parity alone does not replace it.

Use whatever computer-use, GUI, or browser automation is available in the environment. The protocol is tool-agnostic; do not assume a particular vendor or a particular diff script. The requirement is inspected image evidence from the candidate and a usable target reference, not a number from a black box.

## Reference modes

Gate 7 supports two explicit target modes. Declare the mode per comparison in the report.

- **A — live/inspectable reference (preferred):** drive the reference and candidate through equivalent entry paths and states at the same frame, then compare. This is the only mode that can exercise unobserved routes and behavior in the reference.
- **B — static reference captures:** the user supplies reference screenshot(s) for specific surfaces/states/viewports. Drive the candidate through the corresponding path/state and compare it to the supplied image. This proves parity against the supplied captures only; it cannot establish parity for routes, states, or viewports those captures do not show, and it cannot establish live behavior. Rows not evidenced by a supplied capture stay `unknown`/`open` unless the user approved a narrow out-of-scope deviation.

Keep code-level gates before final captures in both modes. In mode B, gate 6 target behavior may come from observed static state evidence or source, but any behavior not evidenced is **not established** and is never assumed. For a task scoped to supplied static UIs, that supports static visual replication for those captures; a requested full interactive clone still needs behavior evidence and is blocked where it is missing.

## Preconditions

- No blocking `unknown` remains in the owning code gates; a bounded estimate may proceed for gate-7 adjudication.
- Mode A: gates 0–6 pass for this surface/state, including gate 6 state/behavior exercises, or only narrow user/product-approved deviations remain. Mode B: gate 6 passes for the evidenced static state coverage and only behavior evidenced by captures, source, or the user's task; unobserved behavior is not established, supports only a static-visual claim, and blocks broader behavioral completion.
- The actual spec was re-extracted after the last candidate change. The target spec remains reference-derived and is refreshed only if the reference evidence, frame, or source changed; retain its provenance.
- Mode A: `frame.yaml` is pinned and both apps can be launched in that frame. Mode B: the candidate can be launched, and at least one user-supplied target capture exists for this exact surface/state/viewport.
- Fonts are loaded in the candidate (and live reference in mode A), animations are disabled or settled, and dynamic data/time is seeded or frozen where the frame requires it.

If a precondition fails, fix it or report the gate blocked for the affected row. Do not capture a mismatched frame and call it close.

## Drive

**Mode A — live reference:** launch reference and candidate in separate windows or contexts at the same viewport, DPR, zoom, theme, locale, and scrollbar mode. Navigate both through the same entry path to the same surface. For nested dialogs, perform the same opening sequence; do not deep-link the candidate to a state the reference reaches by interaction. Drive the same state on both: default, hover (settle the hover), focus, disabled, loading, error, empty, overflow, long content, and each nested dialog/drawer/popover. Use keyboard focus where the reference supports it. Wait for stable layout, then confirm both are still in the intended state immediately before capture.

**Mode B — static reference:** launch and drive the candidate through the corresponding entry path and state. Treat the supplied capture as a fixed target: preserve its pixel dimensions, crop, and content origin. Record what is known and unknown about its viewport, DPR, zoom, theme, locale, fonts, data, and state. Only normalize the candidate to the target when the target's properties are documented or strongly evidenced; never stretch, letterbox, reframe, or recrop images merely to force a match. If the target capture cannot be mapped to a known surface/state/viewport, the row is `unknown`, not compared.

Wait for stable layout: no fonts swapping, no skeleton, no entrance animation, no caret blink in a capture region that matters. If a spinner is part of the state, capture it consistently or mask it as dynamic. Confirm the candidate is still in the intended state immediately before capture.

Capture at least:

- The full surface at the pinned viewport (mode A) or the supplied capture's dimensions (mode B).
- Focused regions: header/toolbar, the primary control group, any dialog body and actions, and one representative long/overflow region.
- Mode A: both apps at the same dimensions; crop neither to hide a difference. Mode B: candidate captures at the supplied target's dimensions without stretching or reframing.
- For responsive rows, the same breakpoint in both (mode A) or a supplied capture at that breakpoint (mode B).

Suggested naming (project-relative):

```
screenshots/ref/<surface>-<state>@<viewport>.png
screenshots/candidate/<surface>-<state>@<viewport>.png
screenshots/cmp/<surface>-<state>@<viewport>-overlay.png
```

## Compare

Use the strongest comparison the environment offers:

- Overlay at 50% opacity or a difference blend to expose shifts.
- Side-by-side at 100% zoom for content and type.
- Pixel or perceptual diff for locating regions, never as the sole verdict.

In mode B, compare the candidate against the supplied target image at the target's dimensions; this is parity against that capture, not a live comparison.

Then inspect every residual region at full size. A diff image tells you where to look; it does not tell you what is wrong. Classify each residual:

| Residual | Route to |
| --- | --- |
| Missing/extra element, wrong parent or order | Gate 2 |
| Offset, size, gap, alignment, overflow, border width | Gate 3 |
| Wrong text, icon, asset, variant, truncation | Gate 4 |
| Wrong color, font, border style, radius, shadow, opacity | Gate 5 |
| Wrong state appearance, missing interaction, or behavior drift | Gate 6 |
| Anti-aliasing, subpixel text, platform font rasterization | Record as rendering difference, not a pass for a layout error |
| Dynamic data, clock, caret, ad, animation frame, cursor | Mask, and list the mask |

After routing a residual to a code gate, fix it, regenerate the actual spec, re-run that gate's diff, and recapture. A screenshot-only patch that changes pixels without correcting the extracted spec is not a fix; it will regress on the next state.

## Masks and dynamic regions

List every mask with its reason and bounds. Typical masks: live timestamps, relative dates, randomized avatars, ads, video, caret, cursor, scrollbar thumb position, loading spinner frame, and OS window chrome outside the app.

Masks are for genuinely dynamic or out-of-scope regions. Never mask a region because it is inconvenient or because it differs. If a region is dynamic but layout-relevant, compare its reserved box and note the content as dynamic.

## Anti-aliasing and rendering noise

Text rasterization, subpixel positioning, GPU shadows, and platform scrollbars can differ between machines even when the code and computed styles match. Account for these by:

- Confirming the computed style and geometry match first.
- Inspecting at 100% and describing the residual (edge weight, hue fringing) rather than assuming it is a layout error.
- Treating noise as an uncertainty entry when it cannot be eliminated, not as a silent pass.

Do not let "anti-aliasing" explain a consistent 2px shift, a wrong font weight, or a different color. Those are real defects.

## Blocked gate

Report the visual gate as **blocked** and do not claim completion when:

- No computer-use/GUI/browser automation is available to drive the candidate.
- Mode A: the reference cannot be launched or accessed in a comparable frame, and no usable supplied target capture exists for the row. If usable static captures do exist, use mode B instead of blocking.
- Mode B: no user-supplied target capture exists for this exact surface/state/viewport, or the supplied capture cannot be mapped to a known frame/surface/state.
- The candidate cannot reach the surface/state that the target capture shows.
- The captures are blank, unaligned, or not the same dimensions after documented normalization attempts.
- Fonts or assets cannot be loaded consistently and the difference is material to the comparison.

A blocked result names the mode, blocker, surfaces/states it affects, and what evidence would unblock it. "Blocked" may be an honest final state of the run and report, but it is never a passed gate, a visual-parity claim, or a completed replication. An unsupported "matches" is not either. Static-capture comparison likewise does not prove behavior or states the captures never show.

## Result format

Record per surface/state:

```markdown
| Surface | State | Viewport | Mode | Capture pair | Masks | Residual class | Route | Result |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| app.settings.account | default | 1280x800 | A live | cmp/account-default-{ref,cand}.png | none | none | — | pass |
| app.settings.account.delete-dialog | error | 1280x800 | B static | cmp/delete-error-{target,cand}.png | timestamp | text shift 1px | gate 3 re-checked | pass against supplied 1280x800 capture; live behavior not evaluated |
```

For a mode B row, a gate-7 pass applies only to that supplied static visual row. It requires every frame property material to the comparison — dimensions, crop/origin, viewport, DPR, zoom, theme, state, fonts, and data as applicable — to be established. Immutable nonmaterial provenance (for example the original OS when visual behavior is otherwise identical) may be noted as a limitation. If a materially needed property is unknown, the row does not pass; it stays `unknown`/`open`. Coverage becomes `covered` only for that static visual row, not for full app behavior.

State explicitly:

- The mode (A live reference or B static capture) and, for B, what the supplied capture does and does not evidence.
- Whether the capture was aligned, non-blank, and same-dimension.
- Which regions were masked and why.
- Which residuals were routed and re-checked.
- Any residual left as rendering noise or accepted deviation, and any state/viewport/behavior not covered by static evidence.

Do not summarize the gate with a bare percentage. If a tool reports one, include it only alongside the inspected residual list and the routed fixes. A high score with an unexamined missing dialog is a failed gate.
