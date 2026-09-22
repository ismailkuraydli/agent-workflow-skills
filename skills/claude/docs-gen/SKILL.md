---
name: docs-gen
description: "Generate and maintain API docs, runbooks, changelogs, READMEs."
---
# Documentation Generator

Generates and maintains project documentation from code. Produces API docs
(OpenAPI/Swagger), runbooks from infrastructure, changelogs from git history,
READMEs from features, and inline doc updates when code changes. Not a one-shot
generator — it maintains docs over time by detecting drift.

## When to Use

- User asks to generate or update documentation
- User says "write the API docs" or "generate the runbook"
- User asks for a changelog
- After a major feature implementation (docs need updating)
- User says "the docs are out of date"
- Periodic documentation review

## Don't Use For

- Architecture spec design (use `architecture-spec`)
- Onboarding guide (use `codebase-onboarding`)
- Release notes for a specific release (use `release-deploy`)

## Prerequisites

- A codebase to document
- `WebSearch` for documentation best practices
- Optional: `openapi` (Python), `swagger-jsdoc` (Node), or framework built-in OpenAPI
- Linear MCP for tracking doc tasks

## Procedure

### Phase 1 — Documentation Audit

1. **Find existing documentation:**
   ```bash
   find . -name "README.md" -o -name "CHANGELOG.md" -o -name "CONTRIBUTING.md" -o -name "RUNBOOK.md" -o -name "API.md" -o -name "docs" -type d 2>/dev/null | grep -v node_modules
   ls docs/ 2>/dev/null
   cat README.md 2>/dev/null | head -30
   ```

2. **Identify what exists vs what's missing:**
   | Doc type | Check | Needed? |
   |---|---|---|
   | README | exists in root? | Always |
   | API docs | OpenAPI spec, route docs? | If API exists |
   | CHANGELOG | exists, up to date? | If releasing |
   | CONTRIBUTING | exists? | If open source |
   | RUNBOOK | ops/incident docs? | If in production |
   | Architecture | ARCHITECTURE.md? | If complex |
   | Inline docs | docstrings/JSDoc on public APIs? | Always |

3. **Detect documentation drift:**
   - Compare README setup section against actual `package.json` scripts
   - Compare API docs against actual route definitions
   - Compare changelog against merged PRs since last entry
   - Compare inline docstrings against function signatures
   - Flag any mismatch as a drift finding

### Phase 2 — API Documentation

4. **Generate OpenAPI spec from code** (if API exists):
   ```bash
   # Express — check for swagger-jsdoc
   grep -rn "swagger\|openapi\|@openapi" --include="*.ts" --include="*.js" . | head -10
   # FastAPI — built-in OpenAPI
   grep -rn "FastAPI(" --include="*.py" . | head -5
   # Go — check for swag/echo-swagger
   grep -rn "swag\|swagger" --include="*.go" . | head -5
   ```

5. **If no OpenAPI spec exists, generate one from route definitions:**
   - Parse all route handlers (from attack surface map in `security-audit`)
   - For each endpoint extract: method, path, params, request body, response body, auth
   - Read the actual handler code to understand request/response shapes
   - Generate OpenAPI 3.1 YAML

6. **For each endpoint, document:**
   - HTTP method and path
   - Authentication requirement
   - Request parameters (path, query, body)
   - Request body schema (with types and validation rules)
   - Response schema (success and error codes)
   - Rate limiting (if applicable)
   - Example request and response

### Phase 3 — README

7. **Generate or update README.md** with:
   - Project name and one-line description
   - What it does (2-3 sentences)
   - Quick start (install, configure, run)
   - Prerequisites (language version, dependencies, services)
   - Configuration (env vars, config files)
   - Development (how to run locally, test, lint, build)
   - Deployment (how to deploy, where it runs)
   - Project structure (key directories)
   - Contributing (link to CONTRIBUTING.md if exists)

8. **Verify README accuracy:**
   - Do the install commands actually work?
   - Are the env vars current? (check `.env.example`)
   - Are the scripts correct? (check `package.json` / `Makefile`)
   - Is the project structure accurate? (check actual directories)

### Phase 4 — CHANGELOG

9. **Generate changelog from git history:**
   ```bash
   # Get merged PRs since last tag
   LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null)
   gh pr list --state merged --base main --limit 50 --json number,title,labels,body,mergedAt
   ```

10. **Format as Keep a Changelog:**
    ```markdown
    ## [Unreleased]

    ### Added
    - New feature X (#PR)

    ### Changed
    - Updated Y behavior (#PR)

    ### Fixed
    - Bug in Z (#PR)

    ### Security
    - Patched CVE-XXXX in dependency (#PR)

    ## [1.4.2] — 2026-09-15
    ...
    ```

11. **If CHANGELOG.md exists:** append new entries at the top under `[Unreleased]`.
    If not: create it with the full history from the first release.

### Phase 5 — Runbook

12. **Generate runbook** (if the system is in production):
    - System overview (what it is, where it runs)
    - Health checks (URLs, expected responses)
    - Common operations (deploy, rollback, restart, scale)
    - Incident procedures (link to `incident-response` postmortems)
    - Environment variables and configuration
    - External dependencies (what they are, how to check their status)
    - Contact info (who to escalate to)

13. **Runbook location:** `docs/runbook.md` or `RUNBOOK.md` in the repo root.

### Phase 6 — Inline Documentation

14. **Check docstrings/JSDoc on public APIs:**
    ```bash
    # Python — functions without docstrings
    grep -rn "^def \|^\s\sdef " --include="*.py" src/ | while read line; do
      file=$(echo "$line" | cut -d: -f1)
      lineno=$(echo "$line" | cut -d: -f2)
      next_line=$(sed -n "$((lineno+1))p" "$file" 2>/dev/null)
      echo "$next_line" | grep -q '\"\"\"\|#\|return\|pass' && echo "MISSING DOCSTRING: $line"
    done
    # TypeScript — exported functions without JSDoc
    grep -rn "export.*function\|export const" --include="*.ts" src/ | grep -v "/**" | head -20
    ```

15. **For each missing docstring:** generate it from the function signature and body:
    - Function name and purpose (inferred from name + body)
    - Parameters (from signature)
    - Return type (from signature or inferred)
    - Example usage
    - Keep it concise — one sentence per parameter, one for purpose, one for return

### Phase 7 — Report and Linear Tickets

16. **Write a documentation report:**
    - What exists, what was generated, what was updated
    - Drift findings (docs that don't match code)
    - Missing documentation list
    - Files created/modified

17. **Create Linear tickets for documentation gaps:**
    - For each missing doc type or drift finding:
      - Title: `docs: [what's missing or drifted]`
      - Description: what's needed, where, and acceptance criteria
      - Priority: 2 (medium) for most docs, 1 (high) for API docs drift
      - Label: "documentation" if available

## Pitfalls

- **Generating docs without verifying accuracy** — a README with wrong install commands is worse than no README. Verify every command.
- **Over-documenting** — internal helper functions don't need docstrings. Document public APIs and complex logic, not everything.
- **Stale docs** — documentation rots. The drift detection in Phase 1 is the most important part — flag mismatches before generating new content.
- **OpenAPI from guessed schemas** — don't guess request/response shapes. Read the actual handler code and types.
- **Changelog without PR numbers** — a changelog entry without a reference is unverifiable. Always include the PR number.
- **Runbook with stale procedures** — if the deploy command changed, the runbook is wrong. Verify against actual infra.
- **Inline docs that restate the code** — "This function returns the user" when the function is `getUser()` is noise. Document WHY, not WHAT.
- **Not updating docs when code changes** — docs-gen should be re-run after major changes. Consider adding it to the post-release checklist.

## Verification

- [ ] Existing documentation audited (what exists, what's missing, what's drifted)
- [ ] API documentation generated or updated (OpenAPI spec + per-endpoint docs)
- [ ] README generated or updated (verified commands are accurate)
- [ ] CHANGELOG updated from merged PRs
- [ ] Runbook generated or updated (if production system)
- [ ] Inline documentation gaps identified and filled for public APIs
- [ ] All drift findings reported
- [ ] Linear tickets created for documentation gaps (if user approved)
