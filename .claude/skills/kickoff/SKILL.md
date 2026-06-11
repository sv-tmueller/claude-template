---
name: kickoff
description: Fan refined, sized GitHub issues out to the agent team. Runs each work package through implement, test, and review to a ready PR, in parallel waves. User-invocable only.
disable-model-invocation: true
argument-hint: <issue numbers | label:<name>>
---

You are the lead and the message bus. Agents cannot call each other; every
handoff is you routing one agent's report into the next agent's task. Keep
your own context lean: delegate the work, route the verdicts, decide the
escalations.

Packages to run: $ARGUMENTS (issue numbers, or `label:<name>` to select by
label).

## 1. Gate

For each issue (`gh issue view <n> --comments`):

- It must be sized. `size:L` or `size:XL` stops kickoff for that issue:
  dispatch the architect for a SPLIT_PROPOSAL, post it on the issue, and
  report it to the user.
- Resume detection: an issue with an open PR or the `in-progress` label is
  resumed, not restarted. Read its sub-plan comment and PR state to find the
  stage it stopped at, and re-enter the pipeline there.
- Dependencies: parse literal `Blocked by: #N` lines in issue bodies. An issue
  whose blocker is not merged waits for a later wave.

## 2. Wave plan

Wave 1 is the issues with no open blockers; wave 2 is the issues blocked only
by wave 1, and so on. Present the plan (issues, sizes, parallelism, expected
PRs) and stop for the user's confirmation. This is the single human gate;
after it, run the wave unattended.

## 3. Per-package pipeline

Run up to 3 packages concurrently; dispatch their agents in parallel. Worktree
isolation keeps packages apart. Within a package the stages are serial:

1. Architect: SUB_PLAN for the issue. Post it as an issue comment. On
   NEEDS_DECISION, park the package for the user and continue the others.
2. Label the issue `in-progress`. Dispatch the developer with the issue number
   and the sub-plan.
3. On DONE: dispatch the tester with the branch and issue number.
4. On FAIL: send the numbered findings verbatim to a fresh developer dispatch
   ("fetch and check out feat/<n>-<slug>, fix exactly these"), then re-test.
5. On PASS: dispatch the reviewer with the PR and issue number.
6. On CHANGES_REQUESTED: same fix loop, then re-test, then re-review.
7. On APPROVE with the full suite green: mark the PR ready (`gh pr ready`),
   remove `in-progress`, post a summary comment on the issue.

Routing rules:

- NEEDS_CONTEXT: answer from the issue, the sub-plan, and the repo docs. If
  you cannot, park the package and queue the question for the user.
- BLOCKED, or the developer pushes back on a reviewer finding: dispatch the
  architect for ARBITRATION and route its outcome.
- Never re-dispatch an unchanged prompt; something in the task must change
  first.
- Cap: 3 fix rounds per stage. On exhaustion, swap `in-progress` for
  `needs-human`, comment the exact state on the PR, and move on to the other
  packages.

## 4. Wave end

Definition of done per package: tester PASS on the full suite, reviewer
APPROVE, PR ready with `Closes #N`, summary comment posted.

Report to the user: PRs ready for review, and packages parked (`needs-human`
or NEEDS_DECISION) with their open questions. The next wave needs this wave
merged, and merging is yours to do, so end with: "merge these PRs, then run
/kickoff again." Reruns are safe; all state lives in GitHub.
