---
name: ask-me
description: Checks whether the current work is blocked on a decision only the user can make and, if so, asks it as a question with ready-made answer options the user can pick. Use when the user invokes /ask-me, or asks things like "is anything waiting on me?", "do you need a decision from me?", "why are you stuck?", "ada keputusan yang harus saya jawab?", "kamu nunggu saya?", "kenapa macet?".
---

# Ask me

The user wants to know whether anything in the current work is waiting on
them. Answer that honestly, and if something is, turn it into questions they
can answer by picking an option.

## 1. Re-check the work

Look back over the current task and conversation for anything that stops
progress or that you parked for the user:

- steps not finished yet, and why;
- errors, failed commands, or tool calls the user denied;
- assumptions you made but never confirmed;
- open choices (design, naming, scope, approach);
- questions you asked earlier that the user never answered;
- instructions that contradict each other;
- if the work is in a git repository with GitHub issues: open issues
  labelled `needs-decision` that no collaborator has answered yet (they were
  raised by an `issues-batch` run and are blocking work). Include each one as
  a decision, with the options the issue lists.

## 2. Keep only real decisions

A decision counts only if **both** are true:

1. It blocks (or will soon block) progress, or proceeding without it risks
   doing the wrong thing.
2. You cannot settle it yourself from the code, docs, project conventions,
   or a sensible default.

Typical decisions that belong to the user:

- actions that are hard to undo or outward-facing: deleting data, force
  pushes, deploys, publishing, sending messages, spending money;
- taste, product, or business choices with no clear right answer;
- credentials, access, or accounts only the user can provide;
- conflicting instructions or requirements;
- scope changes (do more / less than originally asked).

If something has a conventional default, do not ask: pick it and mention the
default you chose in one line.

Never invent a decision just to have something to ask.

## 3a. If nothing is waiting on the user

Reply in one or two lines: say no decision is needed, give a short status of
where the work stands, then continue the work if there is work left.

## 3b. If something is waiting on the user

For each decision:

- State it in one sentence: what is blocked and why.
- Offer **2–4 options**. Put the one you recommend first and append
  `(Recommended)` to its label. Give each option a short description of its
  consequence or trade-off.
- Make every question self-contained: name the files, items, or errors it is
  about, so it can be understood without scrolling back.

Then ask:

- **If the `AskUserQuestion` tool is available**, use it so the options are
  clickable (at most 4 questions per call; the user can always type their own
  answer via "Other", so do not add an "Other" option).
- **Otherwise**, print a numbered list the user can answer briefly, e.g.:

  ```
  1. <decision> — <why it blocks>
     a) <option> (Recommended) — <consequence>
     b) <option> — <consequence>
  2. ...
  Reply with e.g. "1a, 2b".
  ```

If there are more than 4 decisions, ask the most blocking ones first and say
how many remain.

## 4. After the user answers

Continue the work right away according to their choices. Do not ask for
confirmation again on what they already decided.

## Style

- Reply in the language the user is writing in (Indonesian or English).
- Be brief: no long recap of the whole task, only what the decision needs.
