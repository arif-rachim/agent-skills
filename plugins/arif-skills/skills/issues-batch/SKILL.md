---
name: issues-batch
description: Picks a batch of related GitHub issues labelled `ready` (open, unassigned, not in progress) by itself and works it end to end without stopping — claims them (assign to the running GitHub user + `in-progress` label), dispatches each issue to a fresh subagent (independent issues in parallel worktrees), reviews and merges their work onto one branch/PR, makes the recommended decision instead of asking, tests, closes every issue, and waits for CI; anything that cannot be finished becomes a new issue. Use when the user invokes /issues-batch, or asks to "work on the ready issues", "pick up the next issues", "run the sub-issues of #10", "kerjakan issue yang masih terbuka", "kerjakan issue-issue ini sekaligus".
---

# Issues batch

Take a set of related, fully specified issues and work through them until
every one of them is **closed** and the PR's CI is green. The run is
autonomous: after one confirmation at the start, never stop to ask the user.
Decide, record the decision, and keep going. Anything that genuinely cannot be
finished becomes a **new issue**, so the batch always ends with its issues
closed and every leftover problem written down.

You are the **orchestrator**. You select, claim, plan the order, dispatch each
issue to a **worker subagent** with a fresh context, review and integrate the
result, test, and update GitHub. Workers write code; you own GitHub and the
batch branch.

Issues, comments and commit messages are written in **English**. Talk to the
user in their own language.

## Fixed rules for this skill

| Topic | Rule |
| --- | --- |
| Eligible issues | Open, labelled **`ready`**, no assignee, not `in-progress`, not `needs-decision`. |
| Batch size | At most **10** issues per run. |
| Workers | **One fresh subagent per issue**, on a smaller/faster model by default. |
| Parallelism | Independent issues run **in parallel**, each in its own git worktree — **never more than 3 workers at the same time**. |
| Worker budget | At most **15 worker launches per run** in total (first attempts + retries + CI fixes). When the budget is spent, finish remaining work yourself or turn it into `Follow-up:` issues. |
| Git | **One batch branch + one PR**, **one commit per issue**. |
| Claiming | Claim **all** batch issues at the start (assign + `in-progress`). |
| Closing | Close each issue as soon as its tests pass on the batch branch, with a link to the PR. |
| Failure | An issue that cannot be finished is still **closed**, with a link to a new issue describing the problem. |
| CI | Wait for the PR's CI; fix red checks before marking the PR ready for review. |
| Questions | Only one confirmation, at the start. After "go", no questions until the final report. |

## 1. Select the batch

By default you **choose the batch yourself**; the user does not need to name
any issues. `$ARGUMENTS` is optional and only narrows the choice (a parent
issue `#10`, a label, a milestone, or explicit numbers `#11 #12`).

1. Resolve the repository (the user's, otherwise `git remote get-url origin`).
2. Find the GitHub login running this action (GitHub MCP `get_me`, or
   `gh api user --jq .login`).
3. **List eligible issues** (see the table). Pull requests are not issues;
   skip them. If `$ARGUMENTS` was given, keep only eligible issues inside that
   scope. Also check that each one really has Tasks, Definition of Done and
   Testing sections; if not, leave it out and report it as "labelled ready but
   underspecified".
4. **Pick one related group** from the eligible issues, trying these in order
   and taking the first that yields a group:
   1. sub-issues (or task-list items) of the same parent issue — prefer the
      parent whose work is already furthest along, then the oldest parent;
   2. issues in the same milestone;
   3. issues sharing a feature/area label (ignore generic labels such as
      `bug`, `enhancement`, `good first issue`);
   4. issues touching the same files or module, judged from their bodies.

   If no group of two or more exists, take the single oldest eligible issue
   whose dependencies are all closed.
5. Drop issues blocked by an open issue that is outside the group.
6. Take at most 10, dependencies first (`Blocked by #n`, `Depends on #n`),
   then by `[k/n]` prefix, then by issue number.

If there are no eligible issues, tell the user and stop. If open issues exist
but lack the `ready` label, mention how many, and that `issues-breakdown` (or
adding `ready` by hand once an issue is fully specified) makes them eligible.

## 2. Plan the waves

Split the batch into **waves**:

- an issue goes in the first wave whose issues it does not depend on;
- two issues may share a wave only if neither depends on the other **and**
  their Context/Tasks name no common files; otherwise put the later one in a
  later wave;
- at most 3 issues per wave; split bigger waves.

Waves run one after another; issues inside a wave run in parallel.

## 3. Confirm once

Show the user, in a short list: the repository, why these issues form a group
(e.g. "sub-issues of #10"), the batch branch name, and the waves with their
issues (number, title, one line each), plus any issues you left out and why.
Ask for a single "go".

This is the **only** question in the run. Once the user says go, do not ask
anything else until the final report.

## 4. Claim and prepare

1. Make sure the label `in-progress` exists; create it if missing
   (colour `fbca04`, description "Being worked on in a batch run").
2. For **every** issue in the batch: add the running user as assignee, add the
   `in-progress` label, and comment:
   `Started in an issues-batch run on branch \`<branch>\`. Wave w, order k/n.`
3. Create the batch branch from the up-to-date default branch (or use the
   branch the session is required to work on). Name it
   `batch/<parent-or-label>-<short-slug>` when you choose it yourself. Push it.
4. Find the project's real test, lint and type-check commands (issue Testing
   sections, `package.json`, `Makefile`, CI workflows, README) and run the
   test suite once on the batch branch, so you know what was already failing
   before the batch.

## 5. Run each wave

### 5.1 Dispatch workers

For each issue in the wave, start a worker with the Agent tool:

- **Isolation**: give each worker its own git worktree (`isolation:
  "worktree"`), starting from the current tip of the batch branch. With a
  single issue in the wave, a worktree is still preferred so the batch branch
  only receives reviewed work.
- **Model**: a smaller, faster model (e.g. `sonnet`; `haiku` is fine for
  trivial edits). The issues are written so a small model can follow them.
- **Run them in parallel** (all Agent calls for the wave in one message, at
  most 3) and wait for all of them before starting anything else. Retries and
  CI-fix workers count toward the same limit of 3 running at once and toward
  the 15-launch budget.

The worker brief must be self-contained. Include:

- the full issue body (Goal, Context, Tasks, Out of scope, Definition of Done,
  Testing) and any relevant comments;
- the repository conventions and the exact test, lint and type-check commands;
- these rules:
  - work only inside the issue's scope; do not touch files outside it unless
    a task requires it;
  - never ask the user anything; when a choice is needed, pick the option
    most consistent with the issue, existing code and conventions, and the
    easiest to change later — and report it;
  - if something truly needs a human (credentials, money, legal/business
    policy, an irreversible or outward-facing action, contradictory
    requirements), do not do it; finish everything else and report it;
  - never skip, disable, delete or weaken a test; never fake a passing check;
  - run the Testing commands plus lint/type-check; fix and retry up to 3
    times on the same failure;
  - commit the work as **one commit** `<imperative summary> (#<issue>)` on
    the worktree's branch; do not push, do not touch GitHub issues or PRs;
- the report to return, exactly in this shape:

```
Issue: #<n>
Status: done | partial | failed
Branch: <worktree branch>   Commit: <sha>
Changes: <files and one line each>
Decisions: <choice — options considered — why>, or "none"
Needs human: <question, options, recommendation, what is blocked>, or "none"
Tests: <command> → <passed/failed + key output>
Remaining: <tasks not done and why>, or "none"
```

If the Agent tool is not available, do the issues yourself one at a time,
following the same rules.

### 5.2 Review and integrate

For each worker report, in issue order:

1. **Review the diff** of its commit: it matches the Tasks, respects Out of
   scope, meets the Definition of Done, adds the listed tests, and does not
   weaken existing tests.
2. If the review fails or the status is `partial`/`failed` for a reason that
   looks fixable, send the worker your feedback (SendMessage) or start a new
   worker with the feedback. Allow **3 attempts** per issue in total; make the
   last attempt on a stronger model (or do it yourself).
3. **Integrate**: cherry-pick (or merge) the worker's commit onto the batch
   branch. Resolve conflicts yourself, keeping both issues' intent.
4. **Test on the batch branch**: the issue's Testing commands plus lint and
   type-check. Compare with the baseline from step 4.4: only failures the
   batch introduced count.
5. Push the batch branch. Open the PR as a **draft** after the first
   integrated issue if it does not exist yet (title `Batch: <parent or
   label>`), and keep its body listing each issue with its status. Use
   `Refs #n`, not `Closes`, since you close issues yourself.

### 5.3 Record decisions and raise issues

On each issue, comment every decision the worker (or you) made:

```
Decision: <what was chosen>
Options considered: <A>, <B>
Why: <one or two sentences>
```

For each "Needs human" item, create a new issue titled
`Decision needed: <topic>` with the label `needs-decision` (create the label
if missing): context, the exact question, the options with the recommendation,
what is blocked, and a link to the original issue. Comment on the original
issue linking it. Do not add `ready` to these new issues.

If an issue still fails after 3 attempts, create a `Follow-up: <problem>`
issue with: the original issue link, what fails, the exact command and error
output, steps to reproduce, and what was tried. Leave its broken changes out
of the batch branch so the branch stays green, unless the issue explicitly
allows a partial state. Do not add `ready`.

Never perform destructive or outward-facing actions to get unblocked
(deleting data, force-pushing, deploying, sending messages).

### 5.4 Close

When an issue's tests pass on the batch branch, comment a summary, remove
`in-progress`, and close it:

```
Done in <commit sha> on branch `<branch>` — PR <link>.
- Changes: <short list>
- Decisions: <list, or "none">
- Tests: <command> → passed
- Follow-ups: <new issue links, or "none">
```

If an issue could not be fully finished, close it anyway (reason "completed"
if its Definition of Done was met, otherwise "not planned") with the same
summary, stating clearly what is missing and linking the follow-up issue that
now owns it.

### 5.5 Dependent issues

Before starting the next wave, if an issue depends on work that ended up in a
follow-up issue, still dispatch it for the parts that do not depend on it. If
nothing can be done, close it with a comment linking the follow-up issue,
which now lists it as blocked work.

## 6. Finish the batch

1. Run the full test suite on the batch branch and push.
2. **Wait for CI** on the PR (GitHub MCP check runs / `gh pr checks --watch`).
   If a check fails because of the batch, fix it on the batch branch (directly
   or with a worker), push, and wait again — up to **3 rounds**. If it is
   still red, create a `Follow-up:` issue with the failing check and logs,
   comment on the issue(s) that caused it, and leave the PR as a draft.
   Failures that already existed before the batch are reported, not fixed.
3. Update the PR body: each issue with status (closed done / closed with
   follow-up), decisions taken, new issues created, CI status. Mark the PR
   **ready for review** when CI is green.
4. If the batch came from a parent issue and all its sub-issues are now
   closed, comment on the parent with the PR link (leave closing the parent
   to the user unless every Definition of Done item on it is met).
5. Make sure no batch issue is left open or still labelled `in-progress`.
   Clean up worker worktrees and branches.

## 7. Report

Reply to the user with:

- the PR link and its CI status;
- issues closed (fully done vs. closed with follow-up);
- new issues created (`needs-decision` and follow-ups), with links;
- decisions taken on the user's behalf, one line each.

Keep it short; the details live in the issue comments and the PR.

## If the run is interrupted

If you must stop before finishing (out of context, tool failure), for every
batch issue not yet closed: remove `in-progress`, unassign yourself if you
assigned, and comment what was done and what remains. Then report to the user.
