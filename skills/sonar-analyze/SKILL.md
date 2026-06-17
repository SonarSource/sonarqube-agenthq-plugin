---
name: sonar-analyze
description: Analyze a file or code snippet for quality and security issues using SonarQube
argument-hint: "[file-path]"
---

# SonarQube — Code Analysis

Analyze code for quality and security issues using the SonarQube MCP Server.

## Which tool to use

This skill drives one of two MCP tools depending on what your org is entitled to. **Exactly one of them is exposed in a given session** — pick whichever is available:

1. **`mcp__sonarqube__run_advanced_code_analysis`** *(preferred when present).* Runs SonarQube Cloud's server-side engine with the **full project analysis context**, so it performs deeper, cross-file detection. The MCP server reads the file directly from the mounted workspace — you pass a **relative path**, not the file contents. Available only for entitled organizations (the agent mounts the workspace into the server for this).
2. **`mcp__sonarqube__analyze_code_snippet`** *(fallback).* Analyzes **only a single file** at a time with **no project context**, so results may be less complete than a full project scan. You pass the file **contents** inline.

Determine availability by checking which tool exists in your session. If `run_advanced_code_analysis` is available, `analyze_code_snippet` will **not** be — and vice versa. Use the one that is present; do not ask the user to switch tooling.

## Disclaimer

- When using **`analyze_code_snippet`**: results cover a single file with no project context and may not match a full SonarQube project scan. Mention this limitation when presenting results if the user might expect exhaustive coverage.
- When using **`run_advanced_code_analysis`**: results leverage full project context, so no single-file caveat is needed.

## Usage

```
sonar-analyze                        # analyze the file currently in context
sonar-analyze src/auth/login.py      # analyze a specific file
```

## Prerequisites

This skill requires the SonarQube MCP Server to be configured and **one** of the following tools to be available in your session:

- `mcp__sonarqube__run_advanced_code_analysis`, or
- `mcp__sonarqube__analyze_code_snippet`.

If a tool fails due to authentication problems, ask the user to ensure they have given their consent for automatic token exchange through SonarQube Cloud > My Account > Access Tokens > Agent Apps (direct URL is https://sonarcloud.io/account/access-tokens?tab=github_agent_hq). Otherwise surface the tool error verbatim and stop.

## Instructions

### Step 1: Resolve what to analyze

Both tools analyze **one file at a time**. Resolve a single file path:

- If the user provided a file path, use it.
- If no path was provided, look at the current conversation context for a recently mentioned or edited file.
- If nothing is clear, ask: *"Which file would you like me to analyze?"*

Do not accept a directory as input. If the user provides one, ask them to specify a single file.

Use the **project-relative** path (e.g. `src/auth/login.py`), not an absolute path.

### Step 2: Determine the file scope

Always specify the file scope (MAIN or TEST) for more accurate results.

### Step 3: Call the available analysis tool

#### Option A — `run_advanced_code_analysis` (preferred)

The server reads the file itself from the mounted workspace, so **do not** read the file or pass its contents. Provide the relative path, the branch, and the scope:

```json
{
  "projectKey": "<only-if-required>",
  "branchName": "<current branch or PR source branch>",
  "filePath": "src/auth/login.py",
  "fileScope": ["MAIN"]
}
```

- `branchName` selects which analysis context to use. Pass the currently checked-out branch (or, in a PR context, the PR's source branch). If you cannot determine it, omit it and let the server default to the main branch.
- A **default project** is usually configured for this workspace (`SONARQUBE_PROJECT_KEY`), so omit `projectKey` unless the tool requires it or the user targets another project.

#### Option B — `analyze_code_snippet` (fallback)

Read the file's full content first (required for `codeSnippet` and language detection), then detect the **language** from the file extension:

| Extension              | Language key |
| ---------------------- | ------------ |
| `.py`                  | `py`         |
| `.js` `.jsx`           | `js`         |
| `.ts` `.tsx`           | `ts`         |
| `.java`                | `java`       |
| `.go`                  | `go`         |
| `.php`                 | `php`        |
| `.cs`                  | `cs`         |
| `.rb`                  | `rb`         |
| `.swift`               | `swift`      |
| `.kt`                  | `kotlin`     |
| `.c` `.cpp` `.cc` `.h` | `cpp`        |

```json
{
  "projectKey": "<only-if-required>",
  "filePath": "src/auth/login.py",
  "codeSnippet": "<full file content>",
  "language": "py",
  "scope": "MAIN"
}
```

Omit `projectKey` when the integration default applies.

### Step 4: Format the results

**If issues are found**, present them as a table sorted by line number:

```markdown
## SonarQube Analysis — `src/auth/login.py`

Found **3 issue(s)**:

| Line | Severity   | Rule         | Message                                               |
| ---- | ---------- | ------------ | ----------------------------------------------------- |
| 12   | 🔴 Blocker | python:S2077 | Make sure that executing this SQL query is safe here. |
| 34   | 🟠 Major   | python:S1481 | Remove the unused local variable "token".             |
| 67   | 🟡 Minor   | python:S1135 | Complete the task associated to this "TODO" comment.  |
```

Severity icons (the label depends on the server version):
- 🔴 Blocker
- 🟠 Critical / High
- 🟡 Major / Medium
- 🔵 Minor / Low
- ⚪ Info

**If no issues are found**:

```markdown
## SonarQube Analysis — `src/auth/login.py`

✅ No issues found.
```

### Step 5: Next steps

After the results, always add:

- If issues were found: *"Invoke the sonar-fix-issue skill with `<rule> <file>:<line>` to fix a specific issue, or ask me to fix them all."*
- If the user wants to analyze another file: remind them to invoke the sonar-analyze skill with the file relative path.
