# review-plugin

An interactive PR code reviewer for Claude Code, installable via ccmd. Adds the `/review` command.

## What it does

When executed via `/review` in Claude Code, this command performs an interactive code review of a GitHub Pull Request:

1. Fetches PR details, diff, and modified files from GitHub
2. Detects cross-project dependencies (e.g., hub ↔ hub-ui API changes)
3. Invokes the code-reviewer agent to analyze the changes
4. Presents each suggestion interactively — approve, edit, or skip
5. Submits the final review to GitHub only after your explicit confirmation

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with [ccmd](https://github.com/gifflet/ccmd) installed
- GitHub MCP configured with a Personal Access Token that has `repo` scope

## Installation

```bash
ccmd install gifflet/review-plugin
```

## Usage

After installation, use in Claude Code:

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

## How it works

1. `ccmd install` fetches and pins this repository into Claude Code's commands directory
2. Claude Code registers the `/review` command from `commands/review.md`
3. When invoked, Claude Code loads `commands/review.md` into the LLM context
4. The LLM follows the 6-phase workflow defined there:
   - **Phase 1**: Parse and validate input (PR identifier + language)
   - **Phase 2**: Fetch PR details via GitHub MCP
   - **Phase 3**: Analyze cross-project dependencies
   - **Phase 4**: Invoke the code-reviewer agent (`agents/code-reviewer.md`)
   - **Phase 5**: Interactive approval of each suggestion
   - **Phase 6**: Submit review with explicit user confirmation

## Development

The plugin is structured around two key files:

- **`commands/review.md`** — the slash command entry point; defines the 6-phase workflow and orchestrates the review process
- **`agents/code-reviewer.md`** — the specialized agent prompt that performs the actual code analysis and generates review comments

This repository also serves as:
- A production-grade example of Claude Code custom commands
- A reference for building interactive multi-phase slash commands
- A demonstration of GitHub MCP tool integration
- A pattern for commands requiring explicit user confirmation before external actions
