# GitHub Tracking for OCI Skill

> Tracks OCI skill improvements, bug fixes, and feature requests via GitHub Issues.

---

## Repository

**Primary:** `DiazOlveraCo/oci-skill` (public or private)
**Mirror:** `DiazOlveraCo/oracle-troubleshooting` (for operational issues)

---

## Issue Templates for Skill Improvement

### Bug Report (Skill Issue)
```markdown
## Skill Bug Report
**Skill:** oci
**File:** (e.g., references/cli-navigation.md, stages/01-auth/CONTEXT.md)
**Error:** (exact error or incorrect behavior)
**Context:** (what you were doing when it failed)
**Expected:** (what the skill should have done)
**Actual:** (what happened)
**Fix Applied:** (if any manual workaround)
```

### Feature Request (Skill Enhancement)
```markdown
## Skill Feature Request
**Skill:** oci
**Goal:** (what capability should be added)
**Current Gap:** (what's missing or painful now)
**Proposed Solution:** (new reference, stage, command pattern, automation)
**Priority:** (Critical / High / Medium / Low)
**References:** (links to OCI docs, community posts, other skills)
```

### Documentation Update
```markdown
## Skill Documentation Update
**Skill:** oci
**File:** (which reference/stage file)
**Change Type:** (correction / addition / restructuring / deprecation)
**Reason:** (why this change is needed - e.g., new OCI feature, discovered error pattern)
**Source:** (OCI docs URL, session log reference, community discussion)
```

---

## Skill Update Workflow

### When to Create an Issue
- New error pattern discovered (add to `error-research-methodology.md`)
- New CLI command pattern found (add to `cli-navigation.md`)
- New troubleshooting fix (add to `troubleshooting.md`)
- Auth mechanism change (update `callback-mechanism.md`, `persistent-auth.md`)
- New VM access pattern (update `vm-ssh-access.md`)
- MCP config change (update `mcp-config.md`)
- Buddy automation improvement (update `buddy-zoho-vault-auth.md`)
- **Any "known issue" identified during operations — MANDATORY**

### Issue → Fix → Commit Flow
```
1. Encounter error/gap during OCI operations
2. Research root cause (per error-research-methodology.md)
3. Create GitHub issue with template above
4. Implement fix in skill files
5. Test fix in current session
6. Commit with message: "fix(oci): <description> - closes #<issue>"
7. Push to GitHub
```

**Rule:** Steps 3 and 4 are atomic. Do not update skill references without creating the issue first. If you write "known issue" in a reference file, the issue number must be referenced.

### Commit Message Convention
```
fix(oci): <description> - closes #<issue>
feat(oci): <description> - resolves #<issue>
docs(oci): <description>
refactor(oci): <description>
chore(oci): <description>
```

---

## Project Board for Skill Evolution

**Project:** `DiazOlveraCo/oci-skill` → Projects → "OCI Skill Evolution"

### Custom Fields
| Field | Type | Values |
|-------|------|--------|
| Status | Single select | Backlog, Ready, In Progress, Review, Done |
| Type | Single select | Bug, Feature, Docs, Refactor, Research |
| Area | Single select | Auth, CLI, VM, MCP, Buddy, Capacity, Self-Learning |
| Priority | Single select | Critical, High, Medium, Low |
| Effort | Number | 1, 2, 3, 5, 8 (story points) |

### Views
1. **Backlog** — All new issues, grouped by Type
2. **Sprint Board** — Kanban by Status
3. **Research Queue** — Filter: `Type:Research` + `Status:Ready`
4. **High Priority** — Filter: `Priority:Critical,High` + `Status != Done`

---

## Automation

### GitHub Actions for Skill Maintenance
```yaml
# .github/workflows/skill-validation.yml
on:
  push:
    paths:
      - 'oci/**'
  pull_request:
    paths:
      - 'oci/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check SKILL.md structure
        run: |
          # Verify required sections exist
          grep -q "## Self-Learning Loop" oci/SKILL.md
          grep -q "## Dependencies" oci/SKILL.md
          grep -q "## GitHub Issues Workflow" oci/SKILL.md
      - name: Check reference files exist
        run: |
          for f in cli-navigation.md error-research-methodology.md vm-ssh-access.md; do
            test -f "oci/references/$f" || exit 1
          done
```

### Auto-Label by Path
```yaml
# .github/labeler.yml
"area:auth":
  - "oci/references/callback-mechanism.md"
  - "oci/references/persistent-auth.md"
  - "oci/stages/01-auth/**"
"area:cli":
  - "oci/references/cli-navigation.md"
  - "oci/references/error-research-methodology.md"
  - "oci/scripts/Ensure-OCICLI.ps1"
"area:vm":
  - "oci/references/vm-ssh-access.md"
  - "oci/references/reliable-ssh.md"
"area:mcp":
  - "oci/references/mcp-config.md"
  - "oci/stages/03-mcp/**"
"area:buddy":
  - "oci/references/buddy-zoho-vault-auth.md"
"area:capacity":
  - "oci/references/oci-arm-capacity-php.md"
"type:docs":
  - "oci/references/*.md"
"type:code":
  - "oci/stages/**/*.md"
  - "oci/scripts/*.ps1"
"type:self-learning":
  - "oci/scripts/Ensure-OCICLI.ps1"
  - "oci/references/cli-navigation.md"
  - "oci/references/reliable-ssh.md"
  - "oci/stages/01-auth/CONTEXT.md"
```

---

## Skill Health Metrics

Track in GitHub Insights or external dashboard:

| Metric | Target | Source |
|--------|--------|--------|
| Issues resolved per month | ≥ 2 | GitHub Issues closed |
| Time to fix skill bug | < 48h | Issue created → closed |
| Reference files updated per quarter | ≥ 4 | Git commits touching `references/` |
| Self-learning loop triggers | Track all | Issues with `type:self-learning` |
| Skill load success rate | 100% | Session logs (no load failures) |

---

## Integration with Coding Skill

The OCI skill follows the **coding skill's GitHub workflow patterns**:

1. **Issue Lifecycle** — Use coding skill's `Open → Triage → Ready → In Progress → In Review → Done`
2. **PR Templates** — Use coding skill's PR template with OCI-specific sections
3. **Branch Protection** — Require reviews for `main` branch
4. **Merge Queue** — Enable if > 10 PRs/week
5. **Labels** — Use coding skill's taxonomy: `type:`, `status:`, `priority:`, `area:`, `size:`
6. **Projects** — Use coding skill's Project setup with OCI-specific fields

---

## Quick Commands

```bash
# Create skill improvement issue
gh issue create --repo DiazOlveraCo/oci-skill --title "fix(oci): <title>" --body-file .github/ISSUE_TEMPLATE/skill-bug.md

# View skill issues
gh issue list --repo DiazOlveraCo/oci-skill --label "skill:oci"

# View skill project board
gh project view --owner DiazOlveraCo --number <project-number>

# Check skill health
gh api repos/DiazOlveraCo/oci-skill/stats/commit_activity
```