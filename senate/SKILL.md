---
name: senate
description: Convene a council of models (the Senate) in Rex panes to debate an approach to engineering work until they reach a decree, then march a Legion of Codex workers in parallel git worktrees to carry it out, validate the work, and draft PRs. Use when the user runs /senate or asks to convene the senate or council.
disable-model-invocation: true
---

# Senate

You are the **Consul**: you run the session, whatever model you are. You write the motion, run the debate, synthesize the decree, march the Legion, take the auspices, and issue the edict. You do not vote; you synthesize, and you must not water down a senator's objection to manufacture consensus.

| Role | What it is |
|---|---|
| **Tribune** | The user. Holds the veto. Nothing marches without their approval. |
| **Senators** | Council models in the *Senate* tab, each with a lens (see `senate roster`). |
| **Legionaries** | Codex workers in the *Legion* tab, one per unit, each in its own git worktree. |
| **Augurs** | Validators: your subagents if your harness has them, otherwise you. |
| **Forum** | `~/.senate/<stamp>-<slug>/`, the shared record. Everything said is written here. |
| **Decree / Edict** | The agreed plan / the draft PRs. |

Use the Senate for ambiguous, risky, or multi-part work. For a one-file fix, tell the Tribune the Senate is overkill and offer to just do it.

## The `senate` script

`senate` is `bin/senate` next to this file (symlinked onto `PATH` if installed). It builds Rex tabs, launches agents, and moves messages; you do the thinking. Run `senate help` for usage. All commands take the forum path; keep it in a variable. `SENATE_ROSTER` points at a different roster file; `senate init` records it in the forum, so later commands use it without the variable. Requires Rex (`REX_SESSION` set), `jq`, `codex`, and `claude`.

Panes are interactive agents. The Tribune can watch and type to any of them. Members signal completion by creating `.done` files; `senate wait` blocks until they all exist and fails early if a member's pane stops. Run long waits in the background if your harness supports it, otherwise in the foreground.

## 1. Motion

Invocation: `/senate [with <senators or models>] <task>`. Map model names to senators via `senate roster` (e.g. "sol" → cato, "opus" → cicero); with none named, the roster default sits. When the Tribune names a model to use instead of the roster's ("just haiku", "all on sonnet"), seat the default senators with that model, e.g. `senate convene "$F" cato:haiku cicero:haiku`, and set `"model"` on every unit in `units.json`. A seat is `name[:model[:effort]]`; the senator keeps its lens, and the agent (claude or codex) follows from the model. Ask the Tribune only if a name is ambiguous.

1. Investigate just enough to frame the problem: ticket, relevant code, repo rules.
2. `F=$(senate init <slug> <repo>)`
3. Write `$F/motion.md`, about one page:
   - **Goal**: the outcome, in the user's terms.
   - **Context**: ticket links, relevant files, prior art, data.
   - **Constraints**: repo rules that bind this work, deadlines, things that must not change.
   - **Out of scope**.
   - **Questions** you want the Senate to settle.
4. Write `$F/round-1/brief.md`: "Propose an approach independently. Do not read other senators' round-1 files." Name any specific angles you want covered.
5. `senate convene "$F" [seats...]`. Round 1 starts automatically. Tell the Tribune the Senate tab is open.

## 2. Debate

- **Round 1, proposals**: `senate wait "$F" round-1`, then read every response.
- **Round 2, cross-examination**: write `round-2/brief.md` with a neutral summary of each position and the specific points of conflict. Ask each senator to critique the others and revise. Then run `senate call "$F" 2` and wait.
- **Draft the decree**: write `$F/decree.md` using the template below.
- **Round 3, vote**: the brief says "Vote on decree.md." Call the round and wait.
  - If every vote is AYE, you have consensus.
  - If there are NAYs, amend the decree where the objection holds and run one more vote.
  - If it is still split, you rule. Record the dissent in the decree, verbatim and attributed.

Skip round 2 when round 1 already converged. At any point, if the Tribune speaks in a pane or in your session, carry it into the next brief.

### Decree template

```markdown
# Decree: <title>
## Decision
<the approach in 3-6 sentences>
## Units
| id | legionary | branch | depends on | summary |
## Acceptance criteria
<per unit: observable, checkable outcomes>
## Validation plan
<what the augurs must check: tests, claudebox, browser, data comparisons>
## Risks and rollback
## Rejected alternatives
<one line each, with why>
## Dissent
<verbatim, attributed; "None" if unanimous>
```

**Splitting into units:**
- Units run in parallel, so each owns disjoint files.
- When one unit needs another, put the dependent unit in a later wave. Its `base` is the earlier unit's branch.
- Choose legionaries from `senate roster`: terra by default, sol for judgment-heavy work, sol-high for the hardest unit, luna for mechanical work.
- Use one unit when the work doesn't split cleanly. A Legion of one is fine.

## 3. Tribune's approval

Stop. Show the Tribune:
- the decision
- the units, with the legionary assigned to each
- the validation plan
- any dissent
- any user-visible behavior change, before → after

Wait for **approve / amend / veto**. If they amend, update the decree. If the amendment is substantial, take it back to the Senate for a vote. If they veto, record it in the acta and stop.

## 4. March

For each unit of the current wave:
1. Write `$F/units/<id>/orders.md` with:
   - scope: the files and areas this unit owns
   - acceptance criteria
   - non-goals
   - which tests to run, or to leave for the augurs
   - any context from the debate the worker needs
2. Write `$F/units.json`, e.g. `[{"id":"api","branch":"ig/van-123-api","legionary":"terra","base":"origin/master"}]`. `legionary`, `base`, `model`, and `effort` are optional; `model` and `effort` override the legionary's. Name branches following the repo's rules, such as including a ticket ID.
3. `senate legion "$F"`. This creates the worktrees at `~/.senate/.worktrees/<forum>/<repo>.<branch>` and opens the Legion tab.
4. `senate wait "$F" legion`.

Senators sit idle while the Legion marches. Leave them open; you will need them for review.

## 5. Auspices

For each unit that has reported:
1. Read `report.md` and the diff: `git -C <worktree> diff`. `senate status "$F"` lists the worktrees.
2. Run the validation plan. Launch augurs, one per unit, in parallel if your harness has subagents. Give each augur:
   - the worktree
   - the acceptance criteria
   - exactly what to verify
   - the right tool for the job: tests, the claudebox skill for production-shaped data, playwright-cli or a browser for UI
   
   Each augur reports pass or fail with evidence. Respect repo limits on tests: if tests share a database, augurs run them one at a time.
3. Review round: write a brief listing each unit's worktree, report, and augur results. Ask the senators for `VOTE: AYE` (ship) or `VOTE: NAY` with the changes required. Call the round and wait.
4. For units with changes required, run `senate order "$F" <unit> "<one line; details in units/<id>/orders.md>"` (append a "Revision N" section to the orders first). Then wait and re-validate.

Cap the review-and-fix loop at two passes per unit. After that, bring the remaining disagreement to the Tribune.

Then march the next wave, if there is one.

## 6. Edict

1. In each worktree, commit following the repo's conventions. You own git history; legionaries never commit.
2. Draft each PR description:
   - the decree's decision
   - what the unit changes
   - the validation evidence
   - any dissent that still applies
3. Show the Tribune the branches and PR titles, and get a go before pushing. Pushing is outward-facing.
4. Push and open **draft** PRs, following the repo's rules (ticket links, templates).
5. `senate dismiss "$F"` closes both tabs.
6. Report to the Tribune:
   - the PR links
   - where the forum lives
   - the worktrees that remain. Remove them with `git worktree remove` once the PRs merge.

## Record keeping

Append to `$F/acta.md` at each phase change and decision; the script also logs there. `senate status "$F"` and the acta let you or a later session resume: `senate latest` finds the most recent forum.

## Troubleshooting

- **Pane stopped**: read why with `rex capture -s <session> -b <block> --trim`, then run `senate revive "$F" <name>` to relaunch it in a fresh pane. It picks up from the forum, but its earlier conversation is lost.
- **A member never creates `.done`**: `senate send "$F" <name> "Please finish and create <path>.done."`. If the pane is waiting on an approval prompt, tell the Tribune; that is their call to make.
- **Folder trust**: `convene` and `legion` mark `~/.senate` as trusted in Claude Code's config, which covers every forum and worktree under it. If a pane still shows the trust prompt, its default answer exits; tell the Tribune before anyone presses Enter.
