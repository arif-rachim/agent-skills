---
name: grill-me
description: Interrogates an idea, feature or plan the user describes — asks only the questions whose answers would change what gets built, as multiple choice with the simplest option recommended, hard-capped at 2 rounds × 4 questions — then writes a short plan (chat + docs/plans/<slug>.md) ready to pass to issues-breakdown. Use when the user invokes /grill-me, or says "grill me", "poke holes in this", "what am I missing?", "tanya-tanya dulu sebelum saya kerjain", "coba kritisi ide ini".
---

# Grill me

The user describes something they want to build or do. Your job is to find
the gaps that would actually change the plan, ask about them as multiple
choice, and then **stop asking and write the plan**. Bias every question and
every default toward the **simplest thing that works**. The enemy is analysis
paralysis, not an imperfect plan: an issue found later by `issues-breakdown`
or a batch run is cheap; a plan that never ships is not.

Talk to the user in their own language. Write the plan file in **English**
(it will be read by `issues-breakdown` and by small models).

## Hard limits

| Limit | Value |
| --- | --- |
| Rounds of questions | **At most 2.** |
| Questions per round | **At most 4** (one `AskUserQuestion` call per round). |
| Options per question | 2–4, the **simplest viable option first, marked (Recommended)**. |
| Second round | Only if a round-1 answer **changed the shape of the work** and the new gap would block the plan. Otherwise go straight to the plan. |
| Per question | It must pass the test: *"If I don't know this, I will build the wrong thing."* If not, decide it yourself and note the assumption. |

Never ask about things with a conventional default, things the code already
answers, nice-to-haves, or anything that can be changed cheaply later. When
in doubt, **assume the simplest answer, write it in "Assumptions", and move
on**.

## 1. Understand the idea (no questions yet)

Read what the user wrote. If the work is in a repository, spend a few minutes
looking at it before asking anything: existing code that already does part of
this, conventions, how similar features are built, test setup. Many questions
die here because the repo already answers them.

Write down, for yourself, the one-sentence goal and the obvious simplest
implementation. The questions you ask are only the places where that simplest
implementation might be wrong.

## 2. Find the gaps

Walk these areas and keep only gaps that pass the test in Hard limits:

1. **Goal and done** — what does "done" look like; who uses it; is there a
   measurable or visible result?
2. **Scope** — what is explicitly out, what is the smallest version that is
   still useful (v1 vs. later)?
3. **Inputs, data and state** — where data comes from, where it is stored,
   migrations, existing records.
4. **Happy path vs. failure** — what happens on bad input, missing data,
   timeouts, partial failure; what must never happen.
5. **Integration** — what existing code, APIs or services it touches; what
   might break.
6. **Security and access** — who may do this, secrets, permissions, PII.
7. **Testing and verification** — how we will know it works; what can be
   tested automatically.
8. **Rollout** — feature flag, migration order, backwards compatibility,
   anything irreversible.

Pick the **4 most consequential** gaps. If there are fewer than 4, ask fewer.
If there are none, say so and write the plan.

## 3. Ask (round 1)

Ask with `AskUserQuestion` (one call, ≤4 questions). Each question:

- is self-contained: names the feature, file or behaviour it is about;
- says in a few words why it matters ("affects whether we need a migration");
- offers the **simplest viable option first with "(Recommended)"**, then
  1–3 alternatives, each with its trade-off in one line;
- never offers "it depends" or "discuss further" as an option.

If `AskUserQuestion` is unavailable, print the questions as a numbered list
with lettered options and ask the user to reply like `1a, 2c`.

## 4. Round 2 — only if forced

After the answers, ask yourself: did any answer change the shape of the work
so that a new gap now blocks the plan (for example, "multi-tenant" instead
of "single user" raises a data-isolation question)? If yes, ask **up to 4**
more questions, same rules. If no, do not ask anything else — even if you
have "a few more small ones". Those become assumptions.

After round 2 there are **no more questions**, whatever remains.

## 5. Write the plan

Produce the plan in chat **and** save it to `docs/plans/<slug>.md` in the
repository (`<slug>` = short kebab-case name of the feature, prefixed with
the date: `2026-11-05-invoice-csv-export.md`). Create the folder if needed.
Do not commit it unless the user asks; tell them the path.

Template (keep it to about one screen; this is a plan, not a spec):

```markdown
# <Feature name>

**Goal:** one sentence — what and why.
**Done when:** 2–4 verifiable statements.

## Decisions
- <question asked> → <answer chosen> (why in ≤1 line)

## Assumptions (not asked, simplest option taken — change here if wrong)
- ...

## Scope
### In (v1)
- ...
### Out (later or never)
- ...

## Approach
Numbered steps in implementation order, each one roughly one small PR.
Name real files/modules where known.
1. ...
2. ...

## Risks and how we handle them
- <risk> → <mitigation or "accept">

## Verification
- Automated: what tests, where.
- Manual: the one check that proves it works end to end.

## Open questions deferred
Things deliberately not decided now; the implementer picks the simplest
option and records it (or raises a needs-decision issue if truly blocked).
```

## 6. Hand off

Finish with two lines: the path of the plan file, and the offer to run
`issues-breakdown` on it (`/issues-breakdown docs/plans/<slug>.md`). If the
user says yes, invoke that skill with the file path.

## Style

- Short. No recap of the user's idea back to them beyond one sentence.
- Decisive: every open point either becomes a question (max 8 in total) or
  an assumption. Nothing is left as "to be discussed".
- If the user starts going deep on a detail, pull back: note it under
  "Open questions deferred" and continue to the plan.
