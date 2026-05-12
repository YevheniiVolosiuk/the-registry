# The Registry

**Private-first infrastructure for AI skills, agents, and prompts.**

Build once. Sync everywhere. The Registry distributes prompts, skills, and AI agents across your devices, applications, and teams — using simple references to local paths and Git repositories.

No plugins. No marketplace. No lock-in. No build step.
Just portable agent infrastructure.

![The Registry](images/10_meta_skill.svg)

---

## Mental Model

Think of it as **`package.json` for agent capabilities** — but instead of public packages, you point at your own private repos and local paths. Instead of copying skills between projects, you maintain one catalog of references that agents pull on demand.

![Mental Model](images/15_mental_model.svg)

---

## The Problem

![The Problem: Skill Sprawl](images/26_problem_skill_sprawl.svg)

As you build with AI agents, you accumulate skills, custom agents, and prompts — sometimes hundreds of them. They live scattered across repos, devices, and teammates. They duplicate. They drift out of sync. They can't be safely public because they encode your specialized workflows.

![The Problem: Siloed Teams](images/32_problem_team_sharing.svg)

Existing options don't fit:

- **Global `~/.claude/*`** — exposes everything to every agent. Global is the opposite of specialized.
- **Plugin marketplaces** — require manifests, gatekeepers, and platform lock-in.
- **Single monorepo** — doesn't reflect reality. Capabilities are built in the repos where they're used.

## The Solution

![The Solution: The Registry](images/27_solution_library_workflow.svg)

The Registry is a **catalog of references** — pointers to where your agentics actually live (local paths and Git URLs). Nothing is copied or installed until you ask for it.

The workflow is simple: **build → catalog → distribute → use → sync**.

---

## The Agent Is the Runtime

The Registry contains **no traditional application code**.

The entire system is encoded as markdown instructions, catalogs, and workflows — interpreted directly by the agent itself. The agent IS the runtime.

This means:

- **No plugin system** — nothing to register or compile
- **No daemon** — nothing running in the background
- **No runtime installation** — drop the files in, you're done
- **No lock-in** — works with any agent harness that reads markdown skills

Behavior is modified by editing markdown, not code. Any compatible agent harness (Claude Code, Pi, Codex, OpenCode, or anything that loads markdown-based skills) can execute The Registry simply by reading the skill files.

---

## Who This Is For

The Registry is designed for engineers and teams managing specialized AI workflows across multiple repositories, environments, and devices — where capabilities need to stay private, in sync, and easy to distribute.

---

## How It Works

### The Catalog (`registry.yaml`)

```yaml
default_dirs:
  skills:
    - default: .claude/skills/
    - global: ~/.claude/skills/
  agents:
    - default: .claude/agents/
    - global: ~/.claude/agents/
  prompts:
    - default: .claude/commands/
    - global: ~/.claude/commands/

registry:
  skills:
    - name: my-skill
      description: What this skill does
      source: /Users/me/projects/tools/skills/my-skill/SKILL.md
      requires: [agent:helper-agent]
    - name: remote-skill
      description: A skill from a private repo
      source: https://github.com/myorg/private-skills/blob/main/skills/remote-skill/SKILL.md
  agents: []
  prompts: []
```

The catalog stores **pointers, not copies**. Skills live in their source repos. You pull on demand.

### Source Formats

| Format             | Example                                                            |
| ------------------ | ------------------------------------------------------------------ |
| Local filesystem   | `/absolute/path/to/SKILL.md`                                       |
| GitHub browser URL | `https://github.com/org/repo/blob/main/path/to/SKILL.md`           |
| GitHub raw URL     | `https://raw.githubusercontent.com/org/repo/main/path/to/SKILL.md` |

The source points to a specific file; the system pulls the entire parent directory (skills include scripts, references, and assets — not just the markdown file).

For private repos, authentication uses SSH keys or `GITHUB_TOKEN` automatically.

### Typed Dependencies

Dependencies use typed references to avoid name collisions and are resolved recursively before the requested item:

```yaml
requires: [skill:base-utils, agent:reviewer, prompt:task-router]
```

---

## Prerequisites

- **A compatible agent harness** — Claude Code, Pi, or anything that reads `.claude/skills/` style markdown
- **git** — for cloning sources and syncing the catalog
- **gh** *(optional)* — GitHub CLI for forking, cloning, and private repo access. Install: `brew install gh`
- **GitHub SSH key or `GITHUB_TOKEN`** — for private repo access (not needed if using `gh auth login`)
- **just** *(optional)* — for `justfile` shortcuts. Install: `brew install just`

---

## Installation

This is a template repo. You create your own private copy, clone it into your global skills directory, and `/registry` becomes available in every agent session.

### 1. Create Your Private Registry

Create a private copy from this template — this becomes your personal catalog you'll push updates to.

```bash
# Using GitHub CLI
gh repo create new-repo-name --template YevheniiVolosiuk/the-registry
```

Or create it manually via the GitHub UI using the **"Use this template"** button.

### 2. Clone to Global Skills Directory

Clone your repo into `~/.claude/skills/registry`. This path is what makes `/registry` available as a global slash command.

```bash
mkdir -p ~/.claude/skills/registry
git clone <your-repo-url> ~/.claude/skills/registry

# Or using GitHub CLI
gh repo clone <yourname>/the-registry ~/.claude/skills/registry
```

### 3. Configure

Open `~/.claude/skills/registry/SKILL.md` and update the `## Variables` section with your repo URL:

```markdown
- **REGISTRY_REPO_URL**: `https://github.com/yourname/the-registry.git`
```

`REGISTRY_YAML_PATH` and `REGISTRY_SKILL_DIR` are correct by default.

### 4. Verify

Start a new agent session anywhere. `/registry list` should work and show an empty catalog.

---

## Quick Start

![Full Workflow](images/45_solution_full_workflow.svg)

### Add a skill to the catalog

```
/registry add deploy skill from https://github.com/yourorg/infra-tools/blob/main/skills/deploy/SKILL.md
```

This adds a reference to `registry.yaml` and pushes the update to your repo.

### Use it in another project

```
/registry use deploy
```

Pulls the skill from the source into `.claude/skills/deploy/`. Add `install globally` to install to `~/.claude/skills/`.

### Push improvements back

```
/registry push deploy
```

Every device that runs `/registry sync` gets the latest version.

### Sync everything

```
/registry sync
```

---

## Commands

| Command                      | What It Does                                               |
| ---------------------------- | ---------------------------------------------------------- |
| `/registry install`          | First-time setup — create, clone, configure                |
| `/registry add <details>`    | Register a new entry in the catalog                        |
| `/registry use <name>`       | Pull from source into local directory (install or refresh) |
| `/registry push <name>`      | Push local changes back to the source                      |
| `/registry remove <name>`    | Remove from catalog and optionally delete local copy       |
| `/registry list`             | Show full catalog with install status                      |
| `/registry sync`             | Re-pull all installed items from source                    |
| `/registry search <keyword>` | Find entries by name or description                        |

### Justfile Shortcuts

Run registry commands from your terminal without an interactive agent session:

```bash
just list                  # List catalog
just use my-skill          # Pull a skill
just push my-skill         # Push changes back
just add "name: foo, description: bar, source: /path/to/SKILL.md"
just sync                  # Re-pull all installed items
just search "keyword"
```

> Justfile recipes use `--dangerously-skip-permissions` because the agent needs filesystem and git access. Review the `justfile` if you want to change this.

---

## Architecture

```
~/.claude/skills/registry/    # The Registry skill (globally installed)
    SKILL.md                  # Agent instructions — the brain
    registry.yaml             # Your catalog of references
    cookbook/                 # Step-by-step guides for each command
        install.md
        add.md
        use.md
        push.md
        remove.md
        list.md
        sync.md
        search.md
    justfile                  # CLI shorthand for all commands
    README.md                 # This file
```

---

## Design Principles

- **Private-first** — built for specialized, competitive-edge capabilities, not a public marketplace
- **Reference-based** — the catalog stores pointers, not copies
- **Agent-native** — the agent reads markdown and executes; no separate runtime
- **Harness-agnostic** — works with any agent that loads markdown skills
- **Catalog, not manifest** — entries define what's *available*, not what's installed. Pull on demand.

---

## The Agentic Stack

![The Agentic Stack](images/03_agentic_stack.svg)

| Layer            | Purpose                                        |
| ---------------- | ---------------------------------------------- |
| **Skills**       | Raw capabilities — what an agent can do        |
| **Agents**       | Scale, parallelism, specialization             |
| **Prompts**      | Orchestration — coordinate skills and agents   |
| **Justfile**     | Terminal access without an interactive session |
| **The Registry** | Distribution across devices, teams, and agents |
