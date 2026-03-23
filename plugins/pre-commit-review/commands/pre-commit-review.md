---
allowed-tools: Bash(git diff*), Bash(git status*), Bash(git log*), Bash(git blame*), Bash(git show*), Bash(coderabbit:*), Bash(cr:*), Bash(curl:*), Bash(which:*), Read
description: Dual-engine code review (Claude agents + CodeRabbit) on uncommitted changes before committing
disable-model-invocation: false
---

Review all uncommitted changes (staged + unstaged) using TWO independent review engines in parallel, then consolidate findings with cross-validation to maximize coverage and minimize false positives.

The two engines are:
- **Engine A (Claude Agents):** 5 specialized Sonnet agents analyzing different dimensions
- **Engine B (CodeRabbit):** External AI-powered review via `coderabbit review --plain -t uncommitted`

Issues found by BOTH engines have higher confidence. Issues found by only ONE engine are still valuable but scored more carefully.

## Steps

### Phase 1: Setup (sequential)

1. Run `git diff` (unstaged) and `git diff --cached` (staged) to collect all uncommitted changes. If there are no changes at all, stop and inform the user.

2. Run `git status --short` to get the list of modified files. Use a Haiku agent to check if the changes (a) are trivially simple and obviously ok (eg. only whitespace, only version bumps), or (b) are too large to review meaningfully (more than 2000 lines of diff). If trivially simple, inform the user and stop. If too large, warn the user but proceed.

3. Use a Haiku agent to give you a list of file paths to (but not the contents of) any relevant CLAUDE.md files from the codebase: the root CLAUDE.md file (if one exists), as well as any CLAUDE.md files in the directories whose files were modified.

4. Use a Haiku agent to summarize the uncommitted changes and return a brief description of what is being changed.

### Phase 2: Dual-engine review (parallel)

Launch ALL of the following in parallel — the 5 Claude agents AND CodeRabbit at the same time:

**Engine A — 5 Claude Sonnet agents (parallel):**

Pass the full diff output to each agent. Each agent returns a list of issues with: file, line(s), description, reason (eg. CLAUDE.md adherence, bug, historical git context, code comment, architecture).

   a. **Agent #1 (CLAUDE.md compliance):** Audit the changes to make sure they comply with the CLAUDE.md. Note that CLAUDE.md is guidance for Claude as it writes code, so not all instructions will be applicable during code review.
   b. **Agent #2 (Bug scan):** Read the diff, then do a shallow scan for obvious bugs. Avoid reading extra context beyond the changes, focusing just on the changes themselves. Focus on large bugs, and avoid small issues and nitpicks. Ignore likely false positives.
   c. **Agent #3 (Git history context):** Read the git blame and history of the modified files (`git log --oneline -10 -- <file>` and `git blame <file>`), to identify any bugs in light of that historical context.
   d. **Agent #4 (Code consistency):** Read the full current content of each modified file to check if the changes are consistent with the surrounding code patterns and architecture.
   e. **Agent #5 (Code comments):** Read code comments in the modified files, and make sure the changes comply with any guidance in the comments (TODOs, warnings, invariants, etc.).

**Engine B — CodeRabbit (parallel with Engine A):**

   f. Run `coderabbit --version 2>/dev/null` to check if CLI is installed.
      - If NOT installed, auto-install it:
        ```bash
        curl -fsSL https://cli.coderabbit.ai/install.sh | sh
        ```
        Then verify with `coderabbit --version`. If install fails, warn the user and proceed with Engine A only.
      - Check auth with `coderabbit auth status 2>&1`. If not authenticated, inform the user:
        > CodeRabbit installed but not authenticated. Run `! coderabbit auth login` to authenticate, then try again.
        Proceed with Engine A only for this run.
      - If installed and authenticated: run `coderabbit review --plain -t uncommitted` and capture the full output.
      - Parse CodeRabbit output into structured issues with: file, line(s), description, severity (Critical/High/Medium/Low), category (bug/security/performance/style).

### Phase 3: Scoring with cross-validation (parallel per issue)

5. Collect ALL issues from both engines into a unified list. For each issue, tag its source:
   - `[A]` = found only by Claude agents
   - `[B]` = found only by CodeRabbit
   - `[A+B]` = found by both (same file, same or overlapping lines, same category of problem)

6. For each issue, launch a parallel Haiku agent that receives: the diff, the issue description, which engine(s) found it, the list of CLAUDE.md files (from step 3), and any CodeRabbit severity. The agent returns a confidence score 0-100 using this rubric (give this rubric to the agent verbatim):

   a. **0:** Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
   b. **25:** Somewhat confident. This might be a real issue, but may also be a false positive. The agent wasn't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant CLAUDE.md.
   c. **50:** Moderately confident. The agent was able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the changes, it's not very important.
   d. **75:** Highly confident. The agent double checked the issue, and verified that it is very likely it is a real issue that will be hit in practice. The existing approach is insufficient. The issue is very important and will directly impact the code's functionality, or it is an issue that is directly mentioned in the relevant CLAUDE.md.
   e. **100:** Absolutely certain. The agent double checked the issue, and confirmed that it is definitely a real issue, that will happen frequently in practice. The evidence directly confirms this.

   **Cross-validation bonus:** When scoring, the agent MUST apply these adjustments:
   - If tagged `[A+B]` (both engines agree): add +15 to the raw score (capped at 100). Two independent engines finding the same issue is strong signal.
   - If tagged `[A]` or `[B]` (single engine): no adjustment. The issue stands on its own merit.
   - If CodeRabbit marked it as Critical or Security: add +10 to the raw score (capped at 100).

### Phase 4: Consolidation and output

7. **Deduplication:** Group issues that refer to the same file + same or overlapping line range + same category of problem. Keep the best description (prefer the more specific one) and merge sources (eg. `[A]` + `[B]` → `[A+B]`). Use the highest score from the group.

8. **Filter:** Remove any issues with a final score below 70.

9. **Sort:** Order remaining issues by score (highest first), then by file path.

10. **Present the results** directly to the user in the following format:

---

### Pre-commit review

Summary: <brief summary of what the changes do>

**Engines used:** Claude Agents (5) ✓ | CodeRabbit ✓ (or ✗ if unavailable)

Found N issues:

1. **<file>:<line>** — <brief description> `[A+B] score:95`
   Reason: <CLAUDE.md rule / bug / security / historical context>

   ```diff
   <relevant diff snippet, 3-5 lines of context>
   ```

2. **<file>:<line>** — <brief description> `[A] score:82`
   Reason: <reason>

   ```diff
   <relevant diff snippet>
   ```

3. **<file>:<line>** — <brief description> `[B] score:78`
   Reason: <CodeRabbit category — eg. security, performance>

   ```diff
   <relevant diff snippet>
   ```

---

Or, if no issues survived filtering:

---

### Pre-commit review

No significant issues found. Changes look good to commit.

**Engines used:** Claude Agents (5) ✓ | CodeRabbit ✓
**Checked:** CLAUDE.md compliance, bugs, security, git history context, code consistency, comment compliance, performance.
**Cross-validated:** All findings were below threshold after dual-engine analysis.

---

## False positive examples (for scoring agents)

- Pre-existing issues (code that was already like that before these changes)
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (eg. missing imports, type errors, formatting). Assume these will be caught by CI.
- General code quality issues (eg. lack of test coverage, poor documentation), unless explicitly required in CLAUDE.md
- Issues called out in CLAUDE.md but explicitly silenced in the code (eg. lint ignore comment)
- Changes in functionality that are likely intentional
- Issues on lines that the user did not modify

## Rules

- Do NOT check build signal, run builds, or run tests. These are out of scope.
- Do NOT post comments to GitHub. This is a local-only review.
- Use `git diff` and `git diff --cached` as the source of truth for what changed.
- The threshold is 70 (lower than single-engine) because cross-validation already filters noise — a 70+ score after dual-engine analysis is high confidence.
- CodeRabbit CLI is auto-installed if missing. If install or auth fails, degrade to Engine A only with threshold 75.
- Make a todo list first to track progress through the phases.

## Phase 5: Fix and commit (after user approval)

After presenting the results, ask the user if they want to fix the issues found.

If the user says yes:

1. Fix all issues that scored above the threshold. Use an agent or fix directly.
2. **ALWAYS commit the fixes immediately after applying them.** This is critical — uncommitted fixes will be mixed with the original changes and confuse future reviews.
3. Stage ONLY the files that were modified by the fixes (not the user's original changes).
4. Create a commit with the message format:
   ```
   fix: apply pre-commit review fixes

   Issues fixed:
   - <brief description of each fix>
   ```
5. After committing, inform the user that fixes were committed separately from their original changes.

**Why commit immediately:** If fixes are left uncommitted, they blend with the user's original diff. The next review run would see both the original changes AND the fixes as "uncommitted changes", making it impossible to distinguish what was reviewed vs what was fixed. Committing fixes separately keeps the git history clean and auditable.
