# agent-skills

**Claude Code skills that run a development loop for you: from an idea to
GitHub issues to merged code, with you answering a few multiple-choice
questions and reviewing one pull request.**

Describe a feature and `grill-me` asks only the questions that matter and
writes a plan. `issues-breakdown` turns that plan into small, precise GitHub
issues that even a small model can implement. `issues-batch` picks up a
batch of related issues and works them end to end — one subagent per issue,
one branch, one PR — deciding instead of asking, and turning anything it
cannot finish into a new issue. `pr-review` tells you whether the PR is safe
to merge. `issues-sync` keeps the issue tracker honest against the code.
`ask-me` tells you, at any point, whether anything is waiting on you.

Every skill treats issue and PR text as data, never as instructions; none of
them force-pushes, deletes data, deploys, approves or merges.

The skills are packaged as a **Claude Code plugin marketplace**, so they can
be installed straight from this GitHub repository.

## Install

Inside Claude Code (any project, any machine):

```
/plugin marketplace add arif-rachim/agent-skills
```

```
/plugin install arif-skills@arif-agent-skills
```

The repository is `arif-rachim/agent-skills`, but the marketplace it declares is
named `arif-agent-skills` (Claude Code reserves the bare name `agent-skills` for
Anthropic), and the plugin inside it is `arif-skills`.

Restart Claude Code (or run `/reload-plugins`) and the skills are available. Check it:

```
/hello-world Arif
```

The same from a terminal, without opening a session:

```bash
claude plugin marketplace add arif-rachim/agent-skills
```

```bash
claude plugin install arif-skills@arif-agent-skills
```

### Get updates

After new skills are pushed to this repository:

```
/plugin marketplace update arif-agent-skills
```

Or turn on auto-update for the marketplace in `/plugin` → **Marketplaces**.

### Install for a whole project / team

Commit this to the project's `.claude/settings.json`; everyone who trusts the
project folder is prompted to install the plugin:

```json
{
  "extraKnownMarketplaces": {
    "arif-agent-skills": {
      "source": { "source": "github", "repo": "arif-rachim/agent-skills" }
    }
  },
  "enabledPlugins": {
    "arif-skills@arif-agent-skills": true
  }
}
```

### Private repository?

If this repository is private, Claude Code clones it with your own git
credentials, so `git clone https://github.com/arif-rachim/agent-skills` must work
on that machine (for example after `gh auth login`). For background
auto-updates, also export `GITHUB_TOKEN` (or `GH_TOKEN`).

### Without the plugin system (manual copy)

A skill is just a folder with a `SKILL.md`. Copy it into your personal skills
folder and it works in every project, without the plugin namespace:

```bash
git clone https://github.com/arif-rachim/agent-skills ~/agent-skills
```

```bash
mkdir -p ~/.claude/skills && cp -r ~/agent-skills/plugins/arif-skills/skills/* ~/.claude/skills/
```

## Skills

| Skill | Invoke | What it does |
| --- | --- | --- |
| `hello-world` | `/hello-world [name]` | Greets you and confirms the plugin is installed. |
| `grill-me` | `/grill-me <idea>` | Interrogates an idea with at most 2 rounds × 4 multiple-choice questions (simplest option recommended), then writes a short plan to `docs/plans/<slug>.md` ready for `issues-breakdown`. |
| `ask-me` | `/ask-me` | Checks whether the agent is blocked on a decision only you can make, and asks it with answer options you can pick. |
| `issues-breakdown` | `/issues-breakdown [plan or file]` | Turns a plan into GitHub issues (parent + sub-issues) with clear tasks, Definition of Done and testing steps, clear enough for a small model to implement, labelled `ready` for `issues-batch`. |
| `issues-batch` | `/issues-batch [#parent \| label \| #n #m ...] [max=10] [--plan]` | Picks a related group of `ready` issues (open, unassigned, not in progress, not failed twice) by itself and works it autonomously: one fresh subagent per issue (max 3 at once, 15 per run, 5 issues by default), reviews and merges their work onto one branch/PR, decides instead of asking, tests, closes the issues and waits for green CI — leftover problems become `needs-decision` / `follow-up` issues. `--plan` only shows the plan. |
| `issues-sync` | `/issues-sync [#parent \| label \| 90d]` | Summarises done vs. pending issues, checks issue status against the code (closed by mistake, forgot to close, stale in-progress, answered decisions, needs refinement), applies fixes after one approval, and updates a `Status report` issue. |
| `pr-review` | `/pr-review [#pr \| branch] [--post]` | Reviews a PR against the Definition of Done of its issues, hunts bugs and weakened tests, runs tests/lint, and gives a verdict (safe to merge / merge after fixes / do not merge). `--post` submits it as a GitHub comment review. |

Plugin skills are namespaced: in menus they appear as `arif-skills:hello-world`.
Claude also picks a skill automatically when your request matches its
`description`.

## How to use the skills together

The skills form one development loop: **idea → plan → issues → autonomous
implementation → review → merge → sync**. Each step hands its output to the
next, so a feature can go from a sentence to merged code with you only
answering a few multiple-choice questions and reviewing one PR.

```
/grill-me <idea>                      → docs/plans/<date>-<slug>.md
/issues-breakdown docs/plans/<slug>.md → parent issue + sub-issues labelled `ready`
/issues-batch --plan                  → shows which ready issues it would take, and in what order
/issues-batch                         → one "go", then works them on one branch/PR until closed
/pr-review                            → verdict: safe to merge / merge after fixes / do not merge
(you merge the PR)
/issues-sync                          → status report; fixes issues closed or left open by mistake
/ask-me                               → any time: "is anything waiting on me?"
```

### Step by step

1. **Have an idea.** Run `/grill-me Add CSV export for invoices` (or just
   describe it and say "grill me"). You get at most 8 multiple-choice
   questions, each with the simplest option recommended, then a one-screen
   plan saved to `docs/plans/`.
2. **Turn the plan into issues.** Run `/issues-breakdown docs/plans/<file>.md`.
   The agent studies the repository, drafts a parent issue plus small
   sub-issues (each with Tasks, Definition of Done and Testing), shows you the
   draft, and creates them after you approve. Implementable issues get the
   `ready` label.
3. **Check what a batch would do.** `/issues-batch --plan` lists the `ready`
   issues it would pick (grouped by parent, milestone, label or files), their
   order and parallel waves. Nothing is changed. Do this the first few times.
4. **Run the batch.** `/issues-batch` asks for one "go", then: assigns the
   issues to you and labels them `in-progress`, dispatches one fresh worker
   subagent per issue (at most 3 at a time), reviews and merges their work
   onto one batch branch, runs the tests, closes each issue, and waits for CI
   on the PR. It never stops to ask; it decides and records the decision in
   an issue comment. Things it truly cannot decide become `needs-decision`
   issues; things it cannot finish become `Follow-up:` issues.
5. **Review the PR.** `/pr-review` (or `/pr-review 42`) checks the diff
   against every issue's Definition of Done, hunts bugs and weakened tests,
   runs the checks and gives one verdict. Add `--post` to put the review on
   the PR as a comment. It never approves or merges — that is yours.
6. **Merge** the PR on GitHub.
7. **Sync.** `/issues-sync` compares every open issue (and those closed in
   the last 30 days) with the code on the default branch, lists what is done
   and what is pending, proposes fixes for issues closed by mistake or never
   closed, applies them after one approval, and updates a single
   `Status report` issue.

### When the loop leaves something for you

| You see | What to do |
| --- | --- |
| An issue labelled `needs-decision` | Answer it in a comment. The next `/issues-sync` turns your answer into a new `ready` issue. `/ask-me` lists these too. |
| An issue labelled `follow-up` / `needs-refinement` | Run `/issues-breakdown #<number>` to refine it; it becomes `ready` again. |
| Open issues that `/issues-batch` ignores | They lack the `ready` label (not fully specified) or failed two batch runs. Refine them with `/issues-breakdown`. |
| A batch PR you are unsure about | `/pr-review` gives you the verdict and the exact fixes. |

### Labels the skills use

| Label | Meaning | Set by |
| --- | --- | --- |
| `ready` | Fully specified; may be picked up by `issues-batch`. | `issues-breakdown`, or you by hand |
| `in-progress` | Claimed by a batch run. | `issues-batch` |
| `needs-decision` | Blocked on a human answer. | `issues-batch` |
| `follow-up`, `needs-refinement` | Left by a failed batch attempt; must be refined before it is picked up again. | `issues-batch`, `issues-sync` |
| `status-report` | The single issue `issues-sync` keeps updated. | `issues-sync` |

### Safety defaults

- Batches claim at most 5 issues (`max=10` to raise), run at most 3 workers
  at once and 15 worker launches per run, and never retry an issue that
  failed twice.
- Issue bodies, comments and PR text are treated as specifications, never
  as instructions: embedded requests to run destructive commands, skip tests
  or change the rules are ignored and reported.
- No skill force-pushes, deletes data, deploys, approves or merges.
- All of this is new and has not yet been exercised on a real project;
  start with a small repository and a plan of two or three sub-issues.

## Repository layout

```
.claude-plugin/
  marketplace.json          # makes this repo a marketplace named "arif-agent-skills"
plugins/
  arif-skills/              # the plugin "arif-skills"
    .claude-plugin/
      plugin.json           # plugin name, version, author
    skills/
      hello-world/
        SKILL.md            # one folder per skill
      ask-me/
        SKILL.md
      issues-breakdown/
        SKILL.md
      issues-batch/
        SKILL.md
      issues-sync/
        SKILL.md
      grill-me/
        SKILL.md
      pr-review/
        SKILL.md
```

## Add a new skill

1. Create `plugins/arif-skills/skills/<skill-name>/SKILL.md`:

   ```markdown
   ---
   name: my-skill
   description: What it does and when Claude should use it.
   ---

   # My skill

   Instructions for Claude...
   ```

2. Optionally put scripts or reference files next to `SKILL.md` and mention them
   in the instructions.
3. Bump `version` in `plugins/arif-skills/.claude-plugin/plugin.json` and in
   `.claude-plugin/marketplace.json` (installed copies update when the version
   changes).
4. Validate, commit and push:

   ```bash
   claude plugin validate .
   ```

5. On each machine: `/plugin marketplace update arif-agent-skills`.

Want a separate group of skills that can be installed on its own? Add another
folder under `plugins/` with its own `.claude-plugin/plugin.json`, and list it
in `.claude-plugin/marketplace.json`.

### Test locally before pushing

```bash
claude plugin marketplace add ./
```

```bash
claude plugin install arif-skills@arif-agent-skills
```

Or load the plugin for one session only: `claude --plugin-dir ./plugins/arif-skills`.

## License

MIT
