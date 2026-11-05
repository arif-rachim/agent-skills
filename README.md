# agent-skills

Arif Rachim's personal [Claude Code](https://code.claude.com) skills and commands,
packaged as a **Claude Code plugin marketplace** so they can be installed straight
from this GitHub repository.

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
| `ask-me` | `/ask-me` | Checks whether the agent is blocked on a decision only you can make, and asks it with answer options you can pick. |
| `issues-breakdown` | `/issues-breakdown [plan or file]` | Turns a plan into GitHub issues (parent + sub-issues) with clear tasks, Definition of Done and testing steps, clear enough for a small model to implement, labelled `ready` for `issues-batch`. |
| `issues-batch` | `/issues-batch [#parent \| label \| #n #m ...]` | Picks a related group of `ready` issues (open, unassigned, not in progress) by itself and works it autonomously: one fresh subagent per issue (max 3 at once, 15 per run), reviews and merges their work onto one branch/PR, decides instead of asking, tests, closes the issues and waits for green CI — leftover problems become new issues. |
| `issues-sync` | `/issues-sync [#parent \| label \| 90d]` | Summarises done vs. pending issues, checks issue status against the code (closed by mistake, forgot to close, stale in-progress), applies fixes after one approval, and updates a `Status report` issue. |

Plugin skills are namespaced: in menus they appear as `arif-skills:hello-world`.
Claude also picks a skill automatically when your request matches its
`description`.

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
