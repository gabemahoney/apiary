---
name: code-review
description: Perform code review of changed files after task completion. Returns a simple list of improvement work items.
---

## Overview

This skill performs code review on files changed during a Task. 
It returns a list of improvement work items for the caller to review, 
Be thorough but not pedantic - focus on substance over style.

## Parameters

You will receive some instructions on which set of work to review. It might be a Bees ticket idea, or a git worktree.

## Your Mission

Analyze changed code files and return a focused list of actionable improvement work items.
Understand the work context from the user input.
Review all commits and changed files.
- Focus only on source code files. Ignore natural language documentation and unit test code.
If no code files were changed, output "No code files to review" and exit.

### Step 0: Understand project best practices
Find any engineering best practices and architecture documentation and understand them. 
Your job is to provide feedback in any case where the work done deviates from the guidance therein.
**Human Pro Tip**: Place references to you project specific best practices documents in `project/.Claude.md`

### Step 1: Run Linter

If the project has linters/formatters configured (ruff, black, eslint, etc.), run them:
Note any linting issues that should be fixed.

### Step 2: Review Changed Files - Critical Eye

For each changed file, use Read to load it and check for issues across these categories (in addition to any project specific best practice):

#### 1. Dead/Obsolete Code
- Commented-out code that should be removed
- Unused functions, variables, or imports
- Old implementations left behind
- Debugging code (print statements, console.log, TODO comments)

#### 2. Architecture & Design
- Inconsistent interfaces (does this match existing patterns?)
- Inappropriate mixing of concerns (business logic, API, data access should be separated)
- Unnecessary abstractions (YAGNI - You Aren't Gonna Need It)
- Inconsistent patterns with the rest of the codebase

#### 3. Security & Correctness (CRITICAL)

Check for security vulnerabilities:
- Input validation: All user inputs should be validated (Pydantic models, type checks)
- SQL queries: Must use parameterized queries (?, :param), never f-strings
- File paths: Use Path(), validate against workspace
- API keys: Loaded from environment/config, never hardcoded
- Authentication: Proper checks on protected endpoints
- Error messages: No sensitive data in error responses

#### 4. Code Quality
- Long/complex functions (>50 lines, deep nesting >3 levels)
- Repeated code blocks (DRY violations)
- Magic numbers/strings (should be named constants)
- Poor variable/function names (unclear purpose)
- Missing comments for complex logic
- Bare except clauses (anti-pattern)

#### 5. Error Handling
- Bare except clauses (`except:` instead of specific exceptions)
- Resources not properly cleaned up (files/connections should use context managers)
- Missing error handling in critical paths
- Poor error messages (not actionable for users)

#### 6. Performance
- Database queries in loops (N+1 problem)
- Loading entire files into memory (should stream)
- No connection pooling for databases
- Synchronous I/O in async functions
- Missing cache invalidation

### Step 8: Prioritize and Filter

Focus on important issues only:
- **Include:** Security vulnerabilities, logic errors, missing tests, architecture problems
- **Exclude:** Trivial style issues, minor naming nitpicks, personal preferences

Each work item should be:
1. Actionable (can become a standalone Task)
2. Specific (includes file:line where applicable)
3. Important (not trivial)
4. Concise (one line description)
5. Applicable (understand requirements and dont aim for more than is needed)

NOTE: It is expected that many times you will return no important issues.
This is OK. Don't feel obliged to report things. Only report if there is something important.
In fact, if you keep reporting things it will cause an infinite loop which is very bad!

### Step 9: Generate Work Item List

Output a simple numbered list directly in your response:

```markdown
## Code Review Work items

1. Fix SQL injection in transactions.py:85 - use parameterized queries instead of f-strings
2. Add input validation to cache.py:45 endpoint - validate user input format
3. Refactor process_transactions() in llm_categorizer.py:120 - function is 60 lines, extract helper functions
4. Remove commented-out code in llm_categorizer.py:200-210
```

