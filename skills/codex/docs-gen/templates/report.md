## Documentation Report: [Repo Name]

### Documentation Audit

| Doc type | Exists? | Up to date? | Action |
|---|---|---|---|
| README.md | ✓ | ⚠ Setup section outdated | Update |
| API docs (OpenAPI) | ✗ | N/A | Generate |
| CHANGELOG.md | ✓ | ✓ | Append |
| CONTRIBUTING.md | ✗ | N/A | Create (if open source) |
| RUNBOOK.md | ✗ | N/A | Generate (if production) |
| Inline docstrings | Partial | 12 functions missing | Add |

### Drift Findings

| # | Finding | Expected | Actual | Fix |
|---|---|---|---|---|
| 1 | README install command | `npm install` | Says `yarn install` | Update to `npm install` |
| 2 | API docs missing /api/v2 | 15 endpoints documented | 12 endpoints documented | Add 3 v2 endpoints |

### Files Generated/Updated

| Action | File | Description |
|---|---|---|
| Generated | docs/openapi.yaml | OpenAPI 3.1 spec, 15 endpoints |
| Updated | README.md | Fixed install command, added config section |
| Updated | CHANGELOG.md | Added [Unreleased] section with 8 PRs |
| Generated | docs/runbook.md | Production runbook with health checks |
| Updated | src/auth/jwt.ts | Added JSDoc to `generateToken()` |

### Summary
- **Generated:** N files
- **Updated:** N files
- **Drift findings:** N
- **Missing inline docs:** N functions

---
*Report by docs-gen skill — [timestamp]*
