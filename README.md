# Pegasus Skills for AI Agents

A collection of AI agent skills for Django and [SaaS Pegasus](https://www.saaspegasus.com/) projects. Built for developers who want Claude Code (or similar AI coding assistants) to help with Django development, Pegasus upgrades, and SaaS application workflows.

## What are Skills?

Skills are markdown files that give AI agents specialized knowledge and workflows for specific tasks. When you add these to your project, Agents like Claude Code can recognize when you're working on a Django/Pegasus task and apply the right frameworks and best practices.

## Available Skills

| Skill | Description | Triggers |
|-------|-------------|----------|
| [resolve-pegasus-conflicts](skills/resolve-pegasus-conflicts/) | Resolve merge conflicts when upgrading SaaS Pegasus | "merge conflicts," "pegasus upgrade," "upgrade conflicts" |

## Installation

### Option 1: CLI Install (Recommended)

Use the [Skills CLI](https://github.com/vercel-labs/skills) to install skills directly:

```bash
# Install all skills
npx skills add saaspegasus/pegasus-skills

# Install specific skills
npx skills add saaspegasus/pegasus-skills --skill resolve-pegasus-conflicts

# List available skills
npx skills add saaspegasus/pegasus-skills --list

# Install globally (available across all projects)
npx skills add saaspegasus/pegasus-skills --global
```

This automatically installs to your `.claude/skills/` directory (or `~/.claude/skills/` for global installs).

### Option 2: Clone and Copy

Clone the entire repo and copy the skills folder:

```bash
git clone https://github.com/saaspegasus/pegasus-skills.git
cp -r pegasus-skills/skills/* .claude/skills/
```

### Option 3: Git Submodule

Add as a submodule for easy updates:

```bash
git submodule add https://github.com/saaspegasus/pegasus-skills.git .claude/pegasus-skills
```

Then reference skills from `.claude/pegasus-skills/skills/`.

### Option 4: Fork and Customize

1. Fork this repository
2. Customize skills for your specific needs
3. Clone your fork into your projects

## Usage

Once installed, just ask Claude Code to help with Django/Pegasus tasks:

```
"Help me resolve these merge conflicts from the Pegasus upgrade"
→ Uses resolve-pegasus-conflicts skill

"I just merged main into the pegasus branch and have conflicts"
→ Uses resolve-pegasus-conflicts skill
```

You can also invoke skills directly:

```
/resolve-pegasus-conflicts
```

### Skill File Structure

Each skill is a directory containing a `SKILL.md` file:

```
skills/
  skill-name/
    SKILL.md
```

The `SKILL.md` file follows this format:

```markdown
---
name: skill-name
description: One-line description for skill selection
---

# Skill Name

[Full instructions for the AI agent]
```

## License

MIT - Use these however you want.
