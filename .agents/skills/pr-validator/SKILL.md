---
name: pr-validator
description: Validate pull requests created by automated agents. Checks Markdown report formatting, verifies URLs, runs available linters or tests, and posts a validation summary as a PR comment. Use when a new PR is opened by an Oz agent or automation.
license: MIT
---

# PR Validator

Validate automated pull requests for correctness, completeness, and quality. Post a detailed validation summary as a PR comment.

## Prerequisites

### 1. Verify GitHub CLI

```bash
gh auth status
```

### 2. Get PR context

Identify the PR to validate from environment variables or parameters:

```bash
PR_NUMBER=${PR_NUMBER:-$(gh pr list --state open --limit 1 --json number --jq '..number')}
echo "Validating PR #$PR_NUMBER"
```

## Validation Workflow

### Step 1: Fetch PR Details

```bash
gh pr view $PR_NUMBER --json title,body,files,author,labels,headRefName
gh pr diff $PR_NUMBER
```

### Step 2: Markdown Report Validation

If the PR contains a report file (e.g., `AUDIT_REPORT.md`):

1. **Structure check**: Verify report has expected sections
   ```bash
   # Check for required sections
   for section in "Summary" "Outdated Dependencies" "Security Issues" "Recommendations"; do
     if grep -q "## $section" AUDIT_REPORT.md; then
       echo "✅ Section found: $section"
     else
       echo "❌ Missing section: $section"
     fi
   done
   ```

2. **Table formatting**: Verify Markdown tables are properly formatted
   ```bash
   # Check for malformed table rows (pipes count mismatch)
   awk '/^\|/ { n=gsub(/\|/,"&"); if (prev && n != prev) print NR": column mismatch"; prev=n }' AUDIT_REPORT.md
   ```

3. **Empty sections**: Flag any sections with headers but no content

### Step 3: URL Verification

Check all URLs in the PR body and changed files:

```bash
# Extract URLs from report
grep -oP 'https?://[^\s\)]+' AUDIT_REPORT.md | while read url; do
  status=$(curl -o /dev/null -s -w "%{http_code}" --max-time 5 "$url")
  if [ "$status" -ge 400 ]  "$status" -eq 000 ]; then
    echo "❌ Broken URL ($status): $url"
  else
    echo "✅ Valid URL ($status): $url"
  fi
done
```

### Step 4: Code Quality Checks

If the PR includes code changes (not just reports):

**Python files:**
```bash
# Check syntax
python -m py_compile changed_file.py 2>&1
# Check for common issues
python -m pylint --errors-only changed_file.py 2>/dev/null
```

**Node.js files:**
```bash
# Check syntax
node --check changed_file.js 2>&1
```

**Terraform files:**
```bash
# Format check
terraform fmt -check -diff *.tf 2>/dev/null
# Validate
terraform validate 2>/dev/null
```

### Step 5: Dependency File Validation

If dependency files were modified:

- `requirements.txt`: Verify all packages exist on PyPI
- `package.json`: Verify valid JSON structure
- `*.tf`: Verify provider version constraints are valid

### Step 6: Generate Validation Summary

Create a validation comment with the following structure:

```markdown
## 🤖 PR Validation Report

**PR:** #{pr_number} — {pr_title}
**Vali:** Oz PR Validator Agent
**Date:** {current_date}

### Results

| Check | Status | Details |
|-------|--------|---------|
| Report structure | ✅/❌ | {details} |
| Table formatting | ✅/❌ | {details} |
| URL verification | ✅/❌ | {n} valid, {n} broken |
| Code syntax | ✅/❌/⏭️ | {details or "skipped"} |
| Dependency files | ✅/❌/⏭️ | {details or "skipped"} |

### Issues Found

{list of any issues, or "No issues found — PR looks good! ✅"}

### Recommendation

- ✅ **LGTM** — Ready to merge
- ⚠️ **Needs attention** — Minor issues found, review recommended
- ❌ **Blocked** — Critical issues must be fixed before merge
```

### Step 7: Post Comment on PR

```bash
gh pr comment $PR_NUMBER --body "$(cat validation_summary.md)"
```

### Step 8: Set PR Status (optional)

If validation passes:
```bash
gh pr edit $PR_NUMBER --add-label "validated"
```

If validation fails:
```bash
gh pr edit $PR_NUMBER --add-label "needs-review"
```

## Error Handling

- If a validation tool is  and continue
- If PR has no report file, validate only code changes
- If PR is empty or has no changes, flag as suspicious
- Always post a comment, even if validation partially fails

## Deliverable

After completion, provide:
- **Comment Link**: URL to the validation comment on the PR
- **Overall Status**: LGTM / Needs Attention / Blocked
- **Summary**: One-line validation result
