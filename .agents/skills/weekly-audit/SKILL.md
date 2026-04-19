---
name: weekly-audit
description: Audit repositories for outdated dependencies, known CVEs, and security issues. Scans multiple repos, generates a Markdown report, and opens a Pull Request with findings. Use on schedule or manually to maintain dependency hygiene across projects.
license: MIT
---

# Weekly Dependency & Security Audit

Perform a comprehensive audit of repository dependencies and security posture. Generate a detailed Markdown report and open a PR with findings.

## Prerequisites

### 1. Verify GitHub CLI

```bash
gh auth status
```

### 2. Identify target repository

Determine which repository to audit from the current working directory or provided context.

## Audit Workflow

### Step 1: Detect Project Type

Identify the technology stack by checking for:

| File | Stack | Audit Tool |
|------|-------|-----------|
| `requirements.txt` / `pyproject.toml` | Python | `pip-audit`, `pip list --outdated` |
| `package.json` | Node.js | `npm audit`, `npm outdated` |
| `*.tf` | Terraform | Version constraint check, `tfsec` patterns |
| `go.mod` | Go | `go list -m -u all` |
| `Gemfile` | Ruby | `bundle audit` |

### Step 2: Check for Outdated Dependencies

**Python:**
```bash
pip install pip-audit 2>/dev/null
pip-audit -r requirements.txt --format json 2>/dev/null || echo "pip-audit not available"
pip install -r requirements.txt --dry-run 2>/dev/null
```

Alternatively, compare pinned versions against PyPI latest:
```bash
for pkg in $(cat requirements.txt | grep -v "^#" | cut -d'=' -f1); do
  echo "Checking $pkg..."
  pip index versions $pkg 2>/dev/null | head -1
done
```

**Node.js:**
```bash
npm outdated --json 2>/dev/null
npm audit --json 2>/dev/null
```

**Terraform:**
- Check `required_providers` version constraints against Terraform Registry latest
- Flag any provider pinned to major version behind current
- Check for deprecated resource arguments

### Step 3: Security Scan

Scan for common security issues:

1. **Hardcoded secrets**: Search for patterns indicating credentials
   ```bash
   grep -rn "password\|secret\|api_key\|token\|private_key" --include="*.tf" --include="*.py" --include="*.js" --include="*.env" . | grep -v node_modules | grep -v ".git"
   ```

2. **Known CVEs**: Cross-reference dependency versions with known vulnerabilities
   - Python: `pip-audit` output
   - Node: `npm audit` output
   - Terraform: Check for insecure resource configurations (public S3, open security groups)

3. **Terraform-specific security checks**:
   - S3 buckets without encryption or with public access
   - Security groups with 0.0.0.0/0 ingress
   - IAM policies with `*` permissions
   - Missing logging/monitoring configurations
   - Hardcoded values in `default` fields of sensitive variables

### Step 4: Generate Report

Create a comprehensive Markdown report with the following structure:

```markdown
# 🔍 Weekly Dependency & Security Audit

**Repository:** {repo_name}
**Date:** {current_date}
**Agent:** Oz Weekly Audit

## Summary

| Category |ssues Found | Severity |
|----------|-------------|----------|
| Outdated Dependencies | {count} | ⚠️ Medium |
| Known CVEs | {count} | 🔴 High |
| Security Issues | {count} | 🔴 High |

## Outdated Dependencies

| Package | Current | Latest | Behind |
|---------|---------|--------|--------|
| {name} | {current_ver} | {latest_ver} | {major/minor/patch} |

## Known Vulnerabilities (CVEs)

| Package | CVE ID | Severity | Description |
|---------|--------|----------|-------------|
| {name} | {cve_id} | {severity} | {brief_desc} |

## Security Issues

| File | Line | Issue | Recommendation |
|------|------|-------|----------------|
| {file} | {line} | {description} | {fix} |

## Recommendations

1. **Critical** (fix immediately): {list critical items}
2. **High** (fix this sprint): {list high items}
3. **Medium** (plan for next sprint): {list medium items}

***
*Generated automatically by Oz Weekly Audit Agent*
```

### Step 5: Create Branch and PR

```bash
# Create audit branch with timestamp
BRANCH_NAt/weekly-$(date +%Y-%m-%d)"
git checkout -b $BRANCH_NAME

# Write report
# Save the generated report to AUDIT_REPORT.md in repo root

git add AUDIT_REPORT.md
git commit -m "audit: weekly dependency & security report $(date +%Y-%m-%d)

Co-Authored-By: Warp <agent@warp.dev>"
git push -u origin $BRANCH_NAME
```

### Step 6: Open Pull Request

```bash
gh pr create \
  --title "🔍 Weekly Audit Report - $(date +%Y-%m-%d)" \
  --body "$(cat AUDIT_REPORT.md)" \
  --base main \
  --label "audit,automated"
```

If labels don't exist yet, create them:
```bash
gh label create "audit" --color "0E8A16" --description "Automated audit reports" 2>/dev/null
gh label create "automated" --color "1D76DB" --description "Created by Oz agent" 2>/dev/null
```

## Multi-Repository Mode

When auditing multiple repositories, iterate:

```bash
for repo in repo1 repo2 repo3; do
  gh repo clone owner/$repo /tmp/$repo
  cd /tmp/$repo
  # Run Steps 1-6
  cd ..
done
```

## Error Handling

- If a dependency tool is not installed, note it  the report and continue with available tools
- If no issues found, still create a "clean bill of health" report
- If git operations fail, report the error and suggest manual intervention

## Deliverable

After completion, provide:
- **PR Link**: URL to the created pull request
- **Summary**: One-line summary of findings (e.g., "Found 3 outdated deps, 1 CVE, 2 hardcoded secrets")
- **Critical Items**: Any issues requiring immediate attention
