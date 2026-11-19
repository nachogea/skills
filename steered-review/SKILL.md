---
name: steered-review
description: Use when the user asks for a steered code review or wants to walk through a PR, test a suspicion, and decide what should block merging. Supports a one-pass review when the user wants findings immediately.
disable-model-invocation: true
---

# Steered Review

Use the user's hunch as a hypothesis. The review should help them understand the change, verify the paths they care about, and make a merge decision.

Use the repository's review rules. Apply Ponytail Review if available for tracing behavior and proposing the smallest complete fix. This workflow also works on its own.

## 1. Understand the change together

- Identify the PR, diff, or revision under review. Read its description, available linked requirements, and existing review discussion when they matter to the decision. State the intended outcome, the current behavior, and the change in a few sentences.
- Restate the user's suspicion as a checkable question. For example, “Does changing this menu break the legacy form when the flag is off?” If the user gives only a PR link, infer likely risks from the diff and start the review.
- If the user asks for a walkthrough, answer the current question first. Explain the existing flow and where the patch changes it using concrete steps, code references, or a small diagram. Continue at the requested pace. If they ask for a full review, carry the investigation through to a verdict in one pass.

## 2. Follow the risky paths

- Trace shared code into its real consumers. Pick the relevant transition and check both sides. Examples include flag off to on, an old client across a deploy, a failed run resuming, and normal writes arriving during a migration.
- For new storage, an extra dependency, or orchestration, identify the requirement that needs it. Check what the existing path already provides and whether a smaller change preserves the same recovery, correctness, and rollout guarantees.
- For performance or rollout claims, identify the condition that determines real impact. Use an available focused test, version-matched source, or authorized telemetry to check that condition. Keep measured impact distinct from an estimate.
- For each likely blocker, trace the reachable trigger to an observable consequence. Compare with the base where it matters, and try to disprove the claim with an existing guard or a targeted check. State what the evidence establishes and what remains uncertain.

## 3. Make the merge decision

- Lead with findings tied to the reviewed revision. Give each finding a code location, a concrete scenario, the consequence, and a small fix or test. Explain the affected user or operator when it matters.
- Say what should block the merge and why. Treat maintainability suggestions and unrelated existing issues as follow-ups unless the diff makes the risk materially worse. When expected behavior is unresolved, state the product decision needed.
- Before posting, recheck the strongest blocker and the weakest claim you would include. Remove unsupported claims from the proposed review. Tell the user when a follow-up changes the verdict, including when their original suspicion turns out to be fine.
- State the checks actually run and any gap that affects the verdict. Keep the review read-only. Post to GitHub only when the user asks, and use the review action they authorized.

## Example prompts

- `$steered-review PR #123. I think this component also runs with the flag off. Check that path before giving me a verdict.`
- `$steered-review PR #456. Walk me through why this migration needs a new table, then we’ll decide whether it’s justified.`
