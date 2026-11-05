---
name: hello-world
description: Greets the user and confirms that the arif-skills plugin is installed. Use when the user says "hello world", asks to test the skills installation, or invokes /hello-world.
---

# Hello World

This skill exists to prove that skills from the `arif-rachim/agent-skills`
repository are installed and loaded by Claude Code.

When this skill runs:

1. Greet the user with exactly this first line:
   `Hello, World! 👋 The arif-skills plugin is installed and working.`
2. If the user passed arguments (`$ARGUMENTS`), greet them by that name on a
   second line, for example `Nice to meet you, Arif.`
3. Finish with one line naming where the skill came from:
   `Loaded from: arif-rachim/agent-skills → plugins/arif-skills/skills/hello-world`

Keep the reply to those lines. Do not run any tools.
