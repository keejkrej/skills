---
tags: [ui-replication, coverage, settings]
max_turns: 40
timeout_seconds: 1200
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash, computer-use]
---

We're cloning the settings experience of a small reference app. The eval harness supplies a runnable reference build, a runnable candidate build, and their start commands and local URLs externally; they are not repo-relative fixtures. A computer-use/browser automation capability must be enabled so the nested dialogs can be reached. See `evals/README.md`. If a fixture, URL, or that capability is missing, say blocked early and stop rather than inventing screens.

The settings area has a root view, an Account panel, a Preferences tab, and a Delete account flow. Delete opens a confirmation dialog; typing the account name and confirming opens a second nested dialog. Reference and candidate support 1280x800 and 390x844.

Before any implementation work, build the coverage inventory and the target layout specs. Inventory every settings surface and state you can reach, including the root, Account, Preferences, and both nested dialogs, and record each row as planned, covered, open, unknown, deviated, or out-of-scope. Deviated/out-of-scope needs explicit user/product approval; do not self-declare gaps. Include default, disabled, error, empty, overflow, and long-content states where the fixture exposes them. Transcribe the Account panel and both dialogs into the neutral layout schema before you build anything.

Write everything under `.agents/ui-replication/`. Do not modify the candidate for this task. If a surface or state cannot be reached, record that with its reason rather than omitting it.
