# Skills

Install into a repo:

```sh
npx skills add keejkrej/skills --skill decision-record
```

That places the skill at `.agents/skills/decision-record/`. Add this to the repo's `AGENTS.md` or `CLAUDE.md` so the agent runs it without being asked:

```markdown
### Decisions

On a behavior change, a "this looks wrong" report, or a reversal of an accepted choice, follow the `decision-record` skill before calling the work done.
```
