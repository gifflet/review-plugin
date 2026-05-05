# review-plugin

An interactive PR code reviewer for Claude Code, installable via ccmd. Adds the `/review` command and the `/cm` skill.

## What it does

### `/review` — interactive PR code reviewer

When executed via `/review` in Claude Code, this command performs an interactive code review of a GitHub Pull Request:

1. Fetches PR details, diff, and modified files from GitHub
2. Detects cross-project dependencies (e.g., hub ↔ hub-ui API changes)
3. Invokes the `review-plugin:code-reviewer` agent to analyze the changes
4. Presents each suggestion interactively — approve, edit, or skip
5. Submits the final review to GitHub only after your explicit confirmation

### `/cm` — commit artifacts generator

Analyzes the current working tree (`git diff HEAD` + `git status`) and generates:

1. A Conventional Commit message (`feat`, `fix`, `docs`, etc.)
2. A kebab-case branch name with a matching prefix
3. A PR title and 2–4 sentence description

Output language is controlled by the argument (`english`/`en` or `portuguese`/`pt`, default `portuguese`).

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with [ccmd](https://github.com/gifflet/ccmd) installed
- GitHub MCP configured with a Personal Access Token that has `repo` scope

## Installation

```bash
ccmd install gifflet/review-plugin
```

## Usage

After installation, use in Claude Code.

### `/review`

```
/review <PR_NUMBER|PR_URL> [LANGUAGE]
```

**Arguments:**
- `PR_NUMBER|PR_URL` (required): PR number (uses current repo's git remote) or full GitHub URL
- `LANGUAGE` (optional): `english`/`en` (default) or `portuguese`/`pt`

**Examples:**
- `/review 123`
- `/review 123 pt`
- `/review https://github.com/org/repo/pull/123 portuguese`
- `/review https://github.com/org/repo/pull/123 en`

### `/cm`

```
/cm [LANGUAGE]
```

**Arguments:**
- `LANGUAGE` (optional): `english`/`en` or `portuguese`/`pt` (default)

**Examples:**
- `/cm`
- `/cm en`
- `/cm portuguese`

The skill is `disable-model-invocation: true` — Claude will not trigger it automatically; you always invoke it explicitly. Stage or modify files first, then run the skill to get commit/branch/PR artifacts grounded in the live diff.

## How it works

1. `ccmd install` fetches and pins this repository into Claude Code's plugin directory
2. Claude Code registers the `/review` command from `commands/review.md` and the `/cm` skill from `skills/cm/SKILL.md`
3. When `/review` is invoked, Claude Code loads `commands/review.md` and the LLM follows the 6-phase workflow defined there:
   - **Phase 1**: Parse and validate input (PR identifier + language)
   - **Phase 2**: Fetch PR details via GitHub MCP
   - **Phase 3**: Analyze cross-project dependencies
   - **Phase 4**: Invoke the `review-plugin:code-reviewer` agent (`agents/code-reviewer.md`)
   - **Phase 5**: Interactive approval of each suggestion
   - **Phase 6**: Submit review with explicit user confirmation
4. When `/cm` is invoked, Claude Code injects the live `git diff HEAD` and `git status --short` into the skill content (via the `` !`<command>` `` shell-injection syntax) and the LLM produces the three artifacts in the requested language.

## Development

The plugin is structured around three key files:

- **`commands/review.md`** — the `/review` slash command entry point; defines the 6-phase workflow and orchestrates the review process
- **`agents/code-reviewer.md`** — the specialized agent prompt that performs the actual code analysis and generates review comments
- **`skills/cm/SKILL.md`** — the `/cm` skill; self-contained, inlines DevHelper-style instructions and uses dynamic shell injection to pull in the current working-tree state

This repository also serves as:
- A production-grade example of Claude Code custom commands and skills
- A reference for building interactive multi-phase slash commands
- A demonstration of GitHub MCP tool integration
- A pattern for commands requiring explicit user confirmation before external actions
- An example of skill authoring with `disable-model-invocation`, `argument-hint`, `allowed-tools`, and dynamic context injection
