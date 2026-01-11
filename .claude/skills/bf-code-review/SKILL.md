---
name: bf-code-review
description: Review unpushed commits and uncommitted changes for bugs, logic errors, security vulnerabilities, and code quality issues. Use when the user says "review pending", "check my changes", "review commits", "review before push", "code review", or wants to find bugs in staged/unstaged changes. Supports "all" parameter to scan entire project.
allowed-tools: Bash, Read, Grep, Glob, Task, TodoWrite, Write, AskUserQuestion
---

# Code Review

Reviews code for bugs, logic errors, security vulnerabilities, and code quality issues. Can review either pending changes or the entire codebase.

## Quick Start

- Run `/bf-code-review` to start a code review (you'll be asked what to review)
- Run `/bf-fix-issues` to fix issues found in the last review

## Instructions

### Step 1: Ask user review preferences

**You MUST use AskUserQuestion** to ask the user TWO questions before proceeding:

#### Question 1: Review Scope

- **Question**: "What would you like to review?"
- **Header**: "Review scope"
- **Options**:
  - **"Changed files only (Recommended)"** - Review only unpushed commits and uncommitted changes. This is faster and uses fewer tokens.
  - **"Entire codebase"** - Scan ALL source files in the project. Warning: This can be a lengthy process and will use significantly more tokens.

#### Question 2: Confidence Level

- **Question**: "What confidence level of issues should be reported?"
- **Header**: "Confidence"
- **Options**:
  - **"High confidence only (Recommended)"** - Only report issues that are almost certainly bugs or vulnerabilities. Fewer false positives.
  - **"Medium confidence"** - Include issues that are likely problems but may have edge cases where they're intentional. More thorough.
  - **"Low confidence (Everything)"** - Report all potential issues including speculative ones. May include false positives but catches subtle bugs.

### Step 2: Gather files based on user selection

#### If user selected "Changed files only":

Collect all pending changes:

1. **Get unpushed commits**:
   ```bash
   git log @{u}..HEAD --oneline 2>/dev/null || git log origin/$(git branch --show-current)..HEAD --oneline 2>/dev/null || git log origin/master..HEAD --oneline
   ```

2. **Get uncommitted changes** (staged and unstaged):
   ```bash
   git status --short
   git diff --name-only
   git diff --cached --name-only
   ```

3. **Get the actual diff content**:
   ```bash
   # For unpushed commits
   git diff @{u}..HEAD 2>/dev/null || git diff origin/$(git branch --show-current)..HEAD 2>/dev/null || git diff origin/master..HEAD

   # For uncommitted changes
   git diff
   git diff --cached
   ```

Then proceed to Step 3.

#### If user selected "Entire codebase":

Gather all source files:

1. **Identify source code files** using Glob tool with patterns like:
   - `**/*.php` (PHP files)
   - `**/*.ts`, `**/*.tsx` (TypeScript files)
   - `**/*.js`, `**/*.jsx` (JavaScript files)
   - `**/*.py` (Python files)
   - Other relevant extensions for the project

2. **Exclude non-source directories**:
   - `vendor/`, `node_modules/` (dependencies)
   - `storage/`, `cache/`, `.cache/` (generated files)
   - `public/build/`, `dist/`, `build/` (compiled assets)
   - `.git/` (version control)
   - Any other generated or third-party directories

3. **Create a file list** of all source files to review

4. **Note in the report** that this is a full project scan

Then proceed to Step 3.

### Step 3: Create review checklist

Create a todo list to track the review systematically:

1. Identify all files to review
2. Review each file for issues
3. Compile findings into a report
4. Write suggested changes to `suggested-changes.md`

### Step 4: Analyse files for issues

Apply the **confidence level filter** selected by the user when deciding what to report.

#### High Confidence Issues (Always Report)

Issues that are almost certainly bugs or security problems:

- **Definite bugs**: Syntax errors, missing required parameters, calling non-existent methods
- **Clear security vulnerabilities**: SQL injection with raw user input, XSS with unsanitised output
- **Runtime errors**: Null reference on definitely-null values, type errors, infinite loops
- **API misuse**: Wrong number of arguments, incorrect return types

#### Medium Confidence Issues (Report if Medium or Low selected)

Issues that are likely problems but may be intentional in some contexts:

- **Potential null references**: Accessing properties without null checks where null is possible
- **Missing error handling**: No try-catch around operations that could throw
- **Validation gaps**: Validation rules that don't fully cover business requirements (e.g., `required_without` + `nullable` allowing all-null states)
- **Race conditions**: Async operations without proper synchronisation
- **Logic gaps**: Conditional logic that may not handle all cases

#### Low Confidence Issues (Report only if Low selected)

Speculative issues that may be false positives:

- **Code smells**: Overly complex logic, duplicated code, inconsistent naming
- **Performance concerns**: Potential N+1 queries, unnecessary iterations
- **Missing best practices**: No type hints, missing documentation
- **Defensive suggestions**: Additional validation that "might" be needed
- **Style issues**: Things that could be cleaner but work correctly

### Step 5: Read and analyse each file

For each file:

1. Read the full file to understand context
2. For "Changed files only" mode: Focus on the changed lines (from diff)
3. For "Entire codebase" mode: Review the entire file
4. Understand what the code is trying to accomplish
5. Check if the implementation achieves the intent correctly
6. Look for edge cases not handled
7. **Apply the confidence filter** - only include issues that meet the selected threshold

### Step 6: Generate report

Compile findings into a structured report displayed to the user:

```markdown
## Review Summary

**Scope**: X unpushed commits, Y uncommitted files
**Confidence level**: High/Medium/Low
**Files reviewed**: Z
**Issues found**: N (Critical: X, Warnings: Y, Informational: Z)

## Critical Issues

Issues that will likely cause bugs or security problems.

### [File: path/to/file.php]

**Line X-Y**: [Issue title]
- **Type**: Security/Bug/Logic Error
- **Confidence**: High/Medium/Low
- **Description**: What's wrong
- **Impact**: What could happen
- **Suggestion**: How to fix

## Warnings

Issues that may cause problems or indicate code smell.

### [File: path/to/file.tsx]

**Line X**: [Issue title]
- **Type**: Potential Bug/Logic Error
- **Confidence**: High/Medium/Low
- **Description**: What's concerning
- **Suggestion**: Recommended change

## Informational

Minor improvements or style suggestions.

### [File: path/to/file.js]

**Line X**: [Issue title]
- **Type**: Code Quality
- **Confidence**: Low
- **Description**: What could be improved
- **Suggestion**: Recommended change

## Files Reviewed

- [x] path/to/file1.php - 2 issues
- [x] path/to/file2.tsx - No issues
- [ ] path/to/file3.js - Skipped (generated file)
```

**For full codebase scans**, update the scope line:
```markdown
**Scope**: Full codebase scan (all source files)
**Confidence level**: High/Medium/Low
**Files reviewed**: Z
**Issues found**: N (Critical: X, Warnings: Y, Informational: Z)
```

### Step 7: Write suggested changes to todo file

**IMPORTANT**: After generating the report, write all actionable issues to `.claude/skills/bf-code-review/suggested-changes.md` as a todo list.

Use the Write tool to create/overwrite the file with this format:

```markdown
# Code Review - Suggested Changes

Generated: [current date/time]
Branch: [branch name]
Scope: [X commits, Y uncommitted files] OR [Full codebase scan]
Confidence: [High/Medium/Low]

## Todo List

Run `/bf-fix-issues` to fix these issues automatically.

### Critical

- [ ] **[path/to/file:45-48]** SQL Injection vulnerability
  - **Type**: Security
  - **Confidence**: High
  - **Description**: User input is interpolated directly into raw query
  - **Fix**: Use parameterized queries or prepared statements

### Warnings

- [ ] **[path/to/file:23]** Missing null check
  - **Type**: Potential Bug
  - **Confidence**: Medium
  - **Description**: Property accessed without checking if parent object exists
  - **Fix**: Add null check before accessing the property

- [ ] **[path/to/file:67]** Validation allows invalid state
  - **Type**: Logic Error
  - **Confidence**: Medium
  - **Description**: Validation rules allow both fields to be null simultaneously
  - **Fix**: Use required_without_all or add custom validation rule

### Informational

- [ ] **[path/to/file:100]** Consider extracting duplicated logic
  - **Type**: Code Quality
  - **Confidence**: Low
  - **Description**: Same logic is duplicated in two places
  - **Fix**: Extract to a shared helper method or function
```

**File path**: `.claude/skills/bf-code-review/suggested-changes.md`

Each todo item MUST include:
- `[ ]` checkbox (unchecked)
- `**[file:line]**` - exact file path and line number(s)
- Issue title
- Type, Confidence, Description, and Fix fields

**Valid Type values**:
- `Security` - Security vulnerabilities (SQL injection, XSS, etc.)
- `Bug` - Definite bugs that will cause errors
- `Logic Error` - Flawed logic that produces incorrect results
- `Potential Bug` - Code that may cause bugs under certain conditions
- `Code Quality` - Style, maintainability, or best practice issues

### Step 8: Prioritise findings

Order issues by severity in both the report and the todo file:

1. **Critical**: Security vulnerabilities, definite bugs, data loss risks
2. **Warnings**: Potential bugs, logic issues, missing error handling
3. **Informational**: Code quality, style, minor improvements

## Confidence Level Examples

### High Confidence (almost certainly a bug)

```php
// Missing required parameter - will cause PHP Fatal Error
$serviceDesk->isUserClient($user); // Method requires 2 parameters
```

```javascript
// Will throw TypeError - accessing property of undefined
const name = user.profile.name; // user.profile is not checked
```

### Medium Confidence (likely a problem)

```php
// Validation gap - both can be null if both keys present with null values
$rules = [
    'user_id' => 'required_without:email|nullable',
    'email' => 'required_without:user_id|nullable',
];
// Request: {"user_id": null, "email": null} passes but causes runtime error
```

```javascript
// Missing dependency may cause stale closure
useEffect(() => {
    fetchData(userId);
}, []); // userId not in deps - may use stale value
```

### Low Confidence (speculative)

```php
// Could extract to reduce duplication
if ($user->isAdmin()) {
    // 10 lines of logic
}
// Same 10 lines appear in another controller
```

```javascript
// Consider memoizing expensive calculation
const result = items.filter(...).map(...).reduce(...);
// May or may not be a performance issue depending on data size
```

## Best Practices

- For "Changed files only" mode: Focus on the **changed code**, not pre-existing issues
- For "Entire codebase" mode: Review all code in each file
- Consider the **context** of surrounding code
- **Respect the confidence level** selected by the user
- Provide **actionable suggestions**, not vague complaints
- Skip generated files, vendor directories, and lock files
- Don't nitpick style issues that linters would catch
- **Always write to `suggested-changes.md`** even if no issues found (write "No issues found")

## What This Skill Does NOT Do

- Modify source code files (only writes to `suggested-changes.md`)
- Run tests
- Apply fixes automatically

## Example Output

```
## Review Summary

**Scope**: 3 unpushed commits, 2 uncommitted files
**Confidence level**: Medium
**Files reviewed**: 5
**Issues found**: 4 (Critical: 1, Warnings: 2, Informational: 1)

## Critical Issues

### [File: src/services/payment.js]

**Line 45-48**: SQL Injection vulnerability
- **Type**: Security
- **Confidence**: High
- **Description**: User input is interpolated directly into raw query string
- **Impact**: Attackers could execute arbitrary SQL commands
- **Suggestion**: Use parameterized queries or prepared statements

## Warnings

### [File: src/controllers/order.js]

**Line 23**: Missing null check
- **Type**: Potential Bug
- **Confidence**: Medium
- **Description**: Property accessed without checking if parent object exists
- **Impact**: Could throw a null reference error at runtime
- **Suggestion**: Add null check before accessing the property

### [File: src/requests/InviteRequest.php]

**Line 15-18**: Validation allows invalid state
- **Type**: Logic Error
- **Confidence**: Medium
- **Description**: required_without + nullable allows both user_id and email to be null
- **Impact**: Causes runtime error when explode('@', null) is called
- **Suggestion**: Use custom validation or required_without_all with additional check

## Informational

### [File: src/utils/helpers.js]

**Line 100**: Consider extracting duplicated logic
- **Type**: Code Quality
- **Confidence**: Low
- **Description**: Same validation logic appears in multiple files
- **Suggestion**: Extract to a shared utility function

---

Suggested changes written to `.claude/skills/bf-code-review/suggested-changes.md`
Run `/bf-fix-issues` to fix these issues automatically.
```
