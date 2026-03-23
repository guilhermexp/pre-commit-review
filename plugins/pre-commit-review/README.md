# pre-commit-review

A Claude Code plugin that reviews uncommitted changes using **two independent AI engines in parallel**, then cross-validates findings to maximize coverage and minimize false positives.

## Engines

| Engine | What it does |
|--------|-------------|
| **Claude Agents** (5 Sonnet in parallel) | CLAUDE.md compliance, bug scan, git history context, code consistency, code comment compliance |
| **CodeRabbit** | Security, performance, best practices, code quality via `coderabbit review --plain -t uncommitted` |

## How it works

1. Collects all staged + unstaged changes via `git diff`
2. Launches **both engines in parallel** (5 Claude agents + CodeRabbit)
3. Tags each finding by source: `[A]` (Claude only), `[B]` (CodeRabbit only), `[A+B]` (both agree)
4. Scores each issue 0-100 with **cross-validation bonuses**:
   - `[A+B]` (both engines agree): **+15** confidence
   - CodeRabbit Critical/Security: **+10** confidence
5. Deduplicates overlapping issues, filters below threshold
6. Presents consolidated results locally (no GitHub comments, no commits)

## Install

```bash
claude plugin add guilhermexp/pre-commit-review
```

## Usage

```
/pre-commit-review
```

Run it before committing. That's it.

## Requirements

- [Claude Code](https://claude.ai/code)
- [CodeRabbit CLI](https://docs.coderabbit.ai/cli) (optional — degrades gracefully if not installed)

## License

MIT
