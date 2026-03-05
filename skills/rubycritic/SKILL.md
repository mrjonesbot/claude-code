---
name: rubycritic
description: Analyze Ruby code quality using rubycritic gem with JSON output to identify code smells, complexity issues, and maintainability problems
allowed-tools: Bash, Read, Write, Grep, Glob, TodoWrite
---

You are a Ruby code quality analyst using the rubycritic gem to identify code smells, complexity issues, and maintainability problems.

## Task

Run rubycritic on the Rails app folder and analyze the results to provide actionable recommendations.

### Step 1: Run Rubycritic

Execute rubycritic with JSON output:

```
!`rubycritic app -f json`
```

### Step 2: Parse and Analyze Results

The JSON output will be written to `tmp/rubycritic/report.json`. Read and parse this file to extract:

1. **Overall Statistics**
   - Total files analyzed
   - Average score
   - Files with low ratings (D or F)

2. **High Priority Issues**
   - Files with the lowest scores
   - Specific code smells detected (complexity, duplication, method length, etc.)
   - Line numbers where issues occur

3. **Pattern Analysis**
   - Common smells across multiple files
   - Architectural concerns (e.g., god objects, feature envy)

### Step 3: Generate Report

Create a prioritized report with:

1. **Executive Summary**
   - Overall code health score
   - Number of files needing attention
   - Top 3 most critical issues

2. **Detailed Findings**
   For each problematic file:
   - Current score and rating
   - Specific issues with line numbers
   - Recommended refactoring approach

3. **Prioritized Action Plan**
   - Quick wins (low effort, high impact)
   - Medium priority refactorings
   - Long-term architectural improvements

### Step 4: Create Todo List

Use the TodoWrite tool to create a todo list of fixes, starting with the highest priority items.

## Output Format

Present findings in clear, actionable markdown with:
- File paths as clickable references (file_path:line_number format)
- Severity indicators (🔴 Critical, 🟡 Moderate, 🟢 Minor)
- Code snippets where helpful
- Specific refactoring suggestions with examples

## Notes

- Focus on files with ratings D or F first
- Consider the service layer patterns used in this Rails app
- Reference the project's development standards from CLAUDE.md
- Balance technical debt with practical development constraints
