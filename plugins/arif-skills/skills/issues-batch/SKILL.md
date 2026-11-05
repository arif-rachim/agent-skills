---
name: issues-batch
description: Picks a batch of related open, unassigned, not-in-progress GitHub issues by itself (or within a parent/label the user names) and works it end to end without stopping — claims them (assign to the running GitHub user + `in-progress` label), implements them in dependency order on one branch with one commit per issue, makes the recommended decision instead of asking, tests, and closes every issue; anything that cannot be finished becomes a new issue. Use when the user invokes /issues-batch, or asks to "work these issues in a batch", "work on the open issues", "pick up the next issues", "run the sub-issues of #10", "kerjakan issue yang masih terbuka", "kerjakan issue-issue ini sekaligus".
---

# Issues batch

Take a set of related open issues and work through them, in order, until every
one of them is **closed**. The run is autonomous: after one confirmation at the
start, never stop to ask the user. Decide, record the decision, and keep going.
Anything that genuinely cannot be finished becomes a **new issue**, so the batch
always ends with its issues closed and every leftover problem written down.

Issues, comments and commit messages are written in **English**. Talk to the
user in their own language.

## Fixed rules for this skill

| Topic | Rule |
| --- | --- |
| Batch size | At most **10** issues per run. |
| Git | **One branch + one PR** for the whole batch, **one commit per issue**. |
| Claiming | Claim **all** batch issues at the start (assign + `in-progress`). |
| Closing | Close each issue **as soon as its tests pass**, with a link to the PR. |
| Failure | An issue that cannot be finished is still **closed**, with a link to a new issue describing the problem. |
| Questions | Only one confirmation, at the start. After "go", no questions until the final report. |

## 1. Select the batch

By default the agent **chooses the batch itself**; the user does not need to
name any issues. `$ARGUMENTS` is optional and only narrows the choice (a
parent issue `#10`, a label, a milestone, or explicit numbers `#11 #12`).

1. Resolve the repository (the user's, otherwise `git remote get-url origin`).
2. Find the GitHub login running this action (GitHub MCP `get_me`, or
   `gh api user --jq .login`).
3. **List eligible issues**: open, **no assignee**, **not** labelled
   `in-progress`, and not labelled `needs-decision`. Pull requests are not
   issues; skip them. If `$ARGUMENTS` was given, keep only eligible issues
   inside that scope.
4. **Pick one related group** from the eligible issues, trying these in order
   and taking the first that yields a group:
   1. open sub-issues (or task-list items) of the same parent issue — prefer
      the parent whose work is already furthest along, then the oldest parent;
   2. issues in the same milestone;
   3. issues sharing a feature/area label (ignore generic labels such as
      `bug`, `enhancement`, `good first issue`);
   4. issues touching the same files or module, judged from their bodies.

   If no group of two or more exists, take the single oldest eligible issue
   whose dependencies are all closed.
5. Drop issues blocked by an open issue that is outside the group.
6. Order them: issues they depend on first (`Blocked by #n`, `Depends on #n`),
   then by `[k/n]` prefix, then by issue number.
7. Take at most the first 10 in that order; the rest wait for the next run.

If there are no eligible issues, tell the user and stop.

## 2. Confirm once

Show the user, in a short list: the repository, why these issues form a group
(e.g. "sub-issues of #10"), the branch name you will use, and the ordered
issues (number, title, one line each), plus any issues you dropped and why.
Ask for a single "go".

This is the **only** question in the run. Once the user says go, do not ask
anything else until the final report.

## 3. Claim and prepare

1. Make sure the label `in-progress` exists; create it if missing
   (colour `fbca04`, description "Being worked on in a batch run").
2. For **every** issue in the batch: add the running user as assignee, add the
   `in-progress` label, and comment:
   `Started in an issues-batch run on branch \`<branch>\`. Order in batch: k/n.`
3. Create the branch from the up-to-date default branch (or use the branch the
   session is required to work on). Name it `batch/<parent-or-label>-<short-slug>`
   when you choose it yourself.
4. Find the project's real test, lint and type-check commands before starting
   (issue Testing sections, `package.json`, `Makefile`, CI workflows, README).

## 4. Work each issue in order

For each issue, one at a time:

### 4.1 Implement
Read the whole issue (Goal, Tasks, Out of scope, Definition of Done, Testing)
and its comments. Do the Tasks. Stay inside the issue's scope.

### 4.2 When a decision is needed, decide
Do not ask the user. Pick the option you would recommend: the one most
consistent with the issue, the existing code, and project conventions, and the
easiest to change later. Record it as a comment on the issue:

```
Decision: <what was chosen>
Options considered: <A>, <B>
Why: <one or two sentences>
```

### 4.3 When the user truly must answer, raise an issue and continue
Some things cannot be decided by the agent: credentials or access, money,
legal or business policy, an irreversible or outward-facing action (deleting
data, force-pushing, deploying, sending messages), or requirements that
contradict each other. For those:

1. Create a new issue titled `Decision needed: <topic>` with the label
   `needs-decision` (create the label if missing). Body: the context, the
   exact question, the options with your recommendation, what is blocked,
   and a link to the original issue.
2. Comment on the original issue linking the new one.
3. Finish every remaining task that does not depend on that answer.

Never perform destructive or outward-facing actions to get unblocked.

### 4.4 Test
Run the Testing section's commands plus the project's lint and type checks,
and check every Definition of Done item.

If something fails, fix it and run again — up to **3 attempts** at the same
failure. Never skip, disable, delete or weaken a test, and never mark a check
as passing when it is not.

If it still fails after 3 attempts, create a new issue titled
`Follow-up: <problem>` containing: the original issue link, what fails, the
exact command and error output, steps to reproduce, and what you already tried.
Revert or leave out any change that breaks the build so the branch stays green,
unless the issue explicitly allows a partial state.

### 4.5 Commit
Commit the issue's changes as one commit: `<imperative summary> (#<issue>)`.
Push the branch after each commit. Open the batch PR as a draft after the
first commit if it does not exist yet (title `Batch: <parent or label>`), and
keep its body listing each issue with its status.

### 4.6 Close
Comment a summary on the issue, then close it and remove `in-progress`:

```
Done in <commit sha> on branch `<branch>` — PR <link>.
- Changes: <short list>
- Decisions: <list, or "none">
- Tests: <command> → passed
- Follow-ups: <new issue links, or "none">
```

If the issue could not be fully finished, close it anyway (reason
"completed" if its Definition of Done was met, otherwise "not planned") with
the same summary, stating clearly what is missing and linking the follow-up
issue that now owns it.

### 4.7 Dependent issues
If a later issue in the batch depends on work that ended up in a follow-up
issue, still attempt the parts that do not depend on it. If nothing can be
done, close it with a comment linking the follow-up issue, which now lists it
as blocked work. Move on to the next issue.

## 5. Finish the batch

1. Run the full test suite once more on the final branch.
2. Update the PR body: each issue with status (closed done / closed with
   follow-up), decisions taken, new issues created. Use `Refs #n` (not
   `Closes`), since the issues are already closed. Mark the PR ready for
   review when the full suite passes; otherwise leave it as a draft and say
   why in the body.
3. If the batch came from a parent issue and all its sub-issues are now
   closed, comment on the parent with the PR link (leave closing the parent
   to the user unless every Definition of Done item on it is met).
4. Make sure no batch issue is left open or still labelled `in-progress`.

## 6. Report

Reply to the user with:

- the PR link;
- issues closed (fully done vs. closed with follow-up);
- new issues created (`needs-decision` and follow-ups), with links;
- decisions taken on the user's behalf, one line each.

Keep it short; the details live in the issue comments and the PR.

## If the run is interrupted

If you must stop before finishing (out of context, tool failure), for every
batch issue not yet closed: remove `in-progress`, unassign yourself if you
assigned, and comment what was done and what remains. Then report to the user.
