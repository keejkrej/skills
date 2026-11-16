# ui-replication evals

These cases are model-graded workflow exercises for the skill, not self-contained applications. No `./reference` or `./candidate` build lives in this repository. The eval harness supplies the fixtures and the automation capability; the prompt files name the contract, and the harness fills it in at run time.

## Harness contract

Before a case runs, the harness must provide:

- A runnable **reference** build and, where the case edits it, a runnable **candidate** build. Paths, start commands, and local URLs are supplied externally (environment/config), not inferred from repo-relative directories.
- A **computer-use, GUI, or browser automation** capability so the agent can drive both builds, reach nested dialogs, and capture screenshots. Prompts list the generic capability `computer-use`; the harness maps it to its actual tool name (for example a Claude Code computer-use/browser tool). The skill body stays vendor-neutral.
- For the layout case, the candidate repo's existing test tooling/runner is installed and runnable, so a framework-native regression check can be authored and executed. The harness supplies any run command/environment details; it must not require installing new dependencies.
- Read/Glob/Grep/Skill/Write/Edit/Bash for artifact inspection and edits.

These cases are authored for a live reference fixture and require driving it. The skill also supports static-reference captures in gate 7 mode B; a harness that supplies only static captures must not score the live-driving graders as executed successes — the affected rows are blocked infrastructure under these cases.

## Executed vs blocked branches

Executed and blocked infrastructure are different outcomes, not two ways to pass. A blocked run passes only the blocked-infrastructure grader, which requires concrete evidence of a failed fixture/URL/automation precondition and no parity or completion claim. The core artifact/action graders are explicitly N/A for a documented blocked infrastructure run, not passed. A bare "blocked" with no failed precondition, or blocking while the required capability is available, does not earn the blocked grader and must not be used to bypass assertions.

Separate a prerequisite failure from a later code-gate failure and a late gate-7 drive failure. If a fixture, URL, or automation capability is missing before gate work begins, the run stops early with overall status blocked and attempts no gates or captures; it must not report that visual gate 7 failed or passed. If startup succeeds and a later code gate fails, the run does work up to that failure, does not run gate 7 as a pass, and reports the code gate incomplete. If gates 0–6 produced their artifacts and the gate-7 drive or capture then fails, the run reports the visual gate blocked with concrete drive/capture blocker evidence, claims no nonblank matched capture or residual fixes, and does not call that an overall early block. All are not executed success.

Do not compare numeric pass rates across the two branches: a blocked run verifies no UI work, while an executed run verifies artifacts and captures. A high score in one branch is not evidence of skill quality in the other.

## Grading

All current graders are `llm`/`trace` graders: this first pass authors model-graded workflow tests and there is no deterministic runner in this repo. That does not mean a run has no deterministic artifacts: for cases that explicitly request them — currently the layout case — the run must generate framework-native executable parity tests in the candidate repository when its stack supports them, and execute them with existing tooling. Other cases follow their own task and graders; in particular, a no-modify case must not modify its candidate just to add tests. These generated tests are deterministic deliverables judged from the trace, test paths, commands, counts, and results recorded in the gate report; this repo does not install or run them. The LLM graders assess whether the run produced and used those artifacts correctly, not whether the repo itself ships an app test suite.

They require concrete artifact evidence (files, fields, capture paths, trace steps), not fluent narrative. Cases are fixture/capability-dependent; a missing fixture is a valid blocked run, not an executed success.

Deterministic artifact grading in this repository remains a remaining risk. Adding a harness-level schema/test runner is out of scope for this pass.
