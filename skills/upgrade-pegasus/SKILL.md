---
name: upgrade-pegasus
description: Upgrade to the latest version of SaaS Pegasus. Use when the user mentions 'upgrade pegasus,' or 'pegasus upgrade.'
---

# Upgrading Pegasus

## Instructions

This skill helps users upgrade their SaaS Pegasus codebase.

Use the `pegasus-cli` to trigger the upgrade. It can be called via the `pegasus` command in the project's python environment.

You first need to list the user's project and find the right one.

```
pegasus projects list
```

You should be able to compare the project names to what's defined `pegasus-config.yaml` to find the right id.

Then run:

```
pegasus projects push <project_id>
```

If the user specifies to use the latest dev version, pass in `--dev`.

You may need to prompt the user to authenticate first. If necessary, do that via the CLI (`pegasus auth`).

After the push completes:

1. Run `git fetch origin` to see the new branch
2. The branch name will be in the format `pegasus-<version>-<timestamp>` (e.g. `pegasus-2026.2.1.3-1771252378.779356`)
3. `git checkout <branch-name>` to switch to it (it will already be up to date from the push)
4. Merge the user's default branch (usually `main`) into the upgrade branch using your "resolve pegasus conflicts" skill
