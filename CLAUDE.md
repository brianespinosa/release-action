# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

See `README.md` for repository overview and contents.

## Requirements

### Distributed Documentation Structure

This repository **REQUIRES** distributed CLAUDE.md files throughout the directory tree. Each subdirectory with significant functionality must have its own CLAUDE.md file:

- `.github/CLAUDE.md` - GitHub Actions workflow and Dependabot documentation

**When adding new features or directories:**

- Create separate CLAUDE.md files in relevant subdirectories
- Do NOT expand this root CLAUDE.md file with detailed information
- Keep this root file focused on high-level repository overview only
- Distributed files keep context focused and only load relevant information when working in specific parts of the codebase

**README.md vs CLAUDE.md distinction:**

When a directory has a README.md, the CLAUDE.md should reference it. When a directory has both files:

- **README.md** - User-facing documentation (what the component does, detailed explanations, setup instructions, architecture overview)
- **CLAUDE.md** - Claude-specific development guidance (critical warnings, gotchas, common tasks with file:line references, testing patterns)
- **Pattern**: CLAUDE.md should reference README.md for overview/details and focus only on what Claude needs to work safely and effectively

### Documentation Maintenance

When making code changes, you MUST proactively suggest updates to relevant documentation:

- **CLAUDE.md files** - If you add/modify workflows, configuration files, or significant functionality, suggest updates to the relevant CLAUDE.md file in that directory
  - Keep CLAUDE.md files concise (prefer pointers over content duplication)
  - Avoid code snippets that will become outdated
  - Use file:line references to point to authoritative sources
  - Focus on high-level "what/why/how", not detailed descriptions
- **README.md files** - If changes affect user-facing documentation or setup instructions, suggest updates to the relevant README.md file
- **Proactive identification** - After completing code changes, explicitly check if documentation has become outdated and offer to update it

Do not wait for the user to ask about documentation. Identify when your changes have made documentation stale and suggest specific updates.

### Git Workflow

All commit messages MUST follow [Conventional Commits](https://www.conventionalcommits.org/) syntax:

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Common types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`

**NEVER** commit with `--no-verify`. If a pre-commit hook fails, fix the underlying issue before committing. Bypassing hooks defeats the purpose of CI parity at commit time.

**Commit message requirements:**

- Keep messages brief and easy to parse - avoid lengthy descriptions
- Focus on specific, scannable details that help humans quickly identify what a commit contains
- **NEVER** add "Generated with Claude Code" or similar footers to commit messages
- **NEVER** add "Co-Authored-By: Claude" or any co-author footers to commits

**Pull request requirements:**

- **NEVER** add "Co-Authored-By: Claude" or any co-author attribution to PR descriptions
- **NEVER** add "Generated with Claude Code" or similar attribution to PRs
- Keep PR descriptions focused on technical changes, testing, and references

### File Operations and Repository Root

**CRITICAL**: All file creation and modification operations MUST be relative to the git repository root.

- Before creating files or directories, identify the repository root with `git rev-parse --show-toplevel`
- NEVER create files outside the repository boundary - they will not be version controlled
- Use `git status` to confirm new files will be tracked by git

### Technical Review Standards

Always fully evaluate the technical merits of questions and suggestions against relevant sources of truth and documentation.

**Do NOT** use validating phrases like "you're absolutely right" or similar affirmations. Instead:

- Verify claims against actual code, configuration files, and documentation
- Challenge suggestions when they conflict with established patterns or best practices
- Provide objective, evidence-based responses
- Respectfully disagree when warranted, citing specific technical reasons

Technical accuracy and honest assessment are more valuable than agreement.
