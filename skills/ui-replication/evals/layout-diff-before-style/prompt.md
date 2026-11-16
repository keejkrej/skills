---
tags: [ui-replication, diff, structure]
max_turns: 45
timeout_seconds: 1500
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, computer-use]
---

The candidate settings dialog was forked from the reference app, and it looks close but is not right. The eval harness supplies both runnable builds plus their start commands and local URLs externally; they are not repo-relative fixtures. A computer-use/browser automation capability must be enabled for rendered extraction. See `evals/README.md`. If a build, URL, or that capability is missing, say blocked early and stop rather than fabricating artifacts.

Do the comparison properly. Transcribe the reference dialog into the layout schema, then extract the rendered candidate into the exact same schema at 1280x800 and diff them field by field after normalizing both sides to the same units, enums, and color representation. I care about hierarchy, parent/child relationships, sibling order, and how the action row is aligned before any color or shadow polish. Do not infer the candidate layout from its source; render it and extract from the DOM or accessibility tree, and test the extraction adapter against a known fixture before trusting a clean diff.

Fix the candidate until the structural gates pass, regenerate the actual spec after each fix, and keep a defect ledger. If a mismatch is genuinely unobservable, mark it unknown instead of calling it fixed. Write artifacts under `.agents/ui-replication/`.

Also add a framework-specific executable regression check derived from the target layout spec, using the candidate repo's existing test tooling (do not add dependencies or a manifest). It must read the rendered runtime, fail on an empty/nonexistent selection or wrong node count, and reuse the same neutral normalization and tolerance decisions rather than hand-written diverging constants. Do not derive expected values from the candidate implementation or accept a snapshot blindly. Run it and record the test file path, exact command, viewport/environment, counts, and failures in `.agents/ui-replication/diff/gate-report.md`. Resolve and re-run failing checks before style polish or final screenshots.
