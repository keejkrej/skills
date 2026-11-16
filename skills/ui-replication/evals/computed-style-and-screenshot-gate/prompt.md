---
tags: [ui-replication, tokens, screenshot]
max_turns: 45
timeout_seconds: 1500
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, computer-use]
---

We're validating a clone of the reference app's Account settings panel. The eval harness supplies a runnable reference build, a runnable candidate build, and their start commands and local URLs; they are provided externally, not as repo-relative directories. A computer-use/browser automation capability must also be enabled by the harness. See `evals/README.md`. If a fixture, URL, or that capability is missing, say blocked early and stop rather than inventing evidence.

First compare computed styles: extract resolved typography, colors, borders, radii, shadows, spacing, and component states from both rendered apps into tokens.yaml, components.yaml, and states.yaml, and produce a field-level diff. Normalize both sides to the same units and color representation before diffing. Fix real mismatches in the candidate and re-extract. Where a state or transition cannot be observed, mark it unknown; unknowns block completion unless a narrow user-approved deviation covers that exact item.

Only after the code-level gates, including state/interaction and behavior checks, pass, run the final screenshot comparison. Drive both apps with the available computer-use or browser automation through the same route at 1280x800, DPR 2, dark theme, with fonts loaded and data seeded. Capture the full panel plus a focused region of the form and action row, at identical dimensions. Compare with an overlay or side-by-side, list every mask, and route each residual difference back to the relevant code gate and rerun it.

If computer-use or the reference cannot be driven for gate 7 after a valid startup, and gates 0–6 already produced their artifacts, report the visual gate blocked with the concrete drive/capture blocker. Do not claim a nonblank matched capture, do not fabricate residual fixes, and do not claim completion; partial or blank captures are not matched evidence. A late gate-7 block is not an early missing-fixture block and is not a passed gate. Do not report an early missing-fixture stop as a gate-7 result.

Write evidence under `.agents/ui-replication/`.
