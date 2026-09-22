## Security Audit: [Repo Name]

### Attack Surface Map

| Entry Point | Type | Data Source | Auth Required | What It Does |
|---|---|---|---|---|
| POST /api/auth/login | API endpoint | User input | No | Authenticates user, returns JWT |
| POST /api/upload | File upload | User file | Yes | Stores uploaded file |
| /webhook/stripe | Webhook | External service | Signature | Processes payment events |

### Technology Fingerprint

| Layer | Technology | Version |
|---|---|---|
| Runtime | Node.js | 22.x |
| Framework | Express | 4.21.2 |
| Database | PostgreSQL | 16.3 |
| ORM | Drizzle | 0.36.4 |
| Auth | jsonwebtoken | 9.0.2 |
| Validation | zod | 3.24.1 |

---

### Findings by Severity

#### CRITICAL

| # | CWE | OWASP | Finding | File:Line | CVSS | Exploit | Fix |
|---|---|---|---|---|---|---|---|
| 1 | CWE-89 | A03:2021 | SQL injection in search | src/db/search.ts:42 | 9.8 | N/A | Use parameterized query via Drizzle's `sql` template |
| 2 | CWE-798 | A07:2021 | Hardcoded API key | src/config.ts:15 | 9.1 | N/A | Move to env var, rotate key |

#### HIGH

| # | CWE | OWASP | Finding | File:Line | CVSS | Fix |
|---|---|---|---|---|---|---|
| 1 | CWE-285 | A01:2021 | No auth check on admin route | src/routes/admin.ts:8 | 7.5 | Add auth middleware |

#### MEDIUM

| # | CWE | OWASP | Finding | File:Line | CVSS | Fix |
|---|---|---|---|---|---|---|
| 1 | CWE-693 | A05:2021 | Missing helmet/secure headers | N/A | 5.3 | Install and configure helmet |

#### LOW

| # | CWE | Finding | File:Line | Fix |
|---|---|---|---|---|
| 1 | CWE-209 | Error message leaks stack trace | src/middleware/error.ts:20 | Strip stack in production |

---

### Recent Vulnerabilities (CVEs)

| CVE | Dependency | Installed | Vulnerable Range | Exploit? | Status | Fix |
|---|---|---|---|---|---|---|
| CVE-2026-1234 | jsonwebtoken | 9.0.2 | <9.1.0 | Public PoC | VULNERABLE | Upgrade to 9.1.0 |
| CVE-2026-5678 | express | 4.21.2 | <4.22.0 | No | NOT AFFECTED | Code doesn't use vulnerable function |

---

### Best Practices Gap Analysis

| Practice | Framework Support | Enabled? | Recommendation |
|---|---|---|---|
| CSRF protection | csurf middleware | ✗ | Add csurf to state-changing routes |
| Rate limiting | express-rate-limit | ✗ | Add to /auth/login, /auth/signup |
| Secure headers | helmet | ✓ | Already configured correctly |
| CORS | cors middleware | ⚠ | Allowed origin is wildcard — restrict to known domains |
| Input validation | zod | ✓ | Used on all API endpoints |
| Parameterized queries | Drizzle ORM | ✓ | All queries use Drizzle's query builder |
| Cookie security | express-cookie | ⚠ | httpOnly set, secure flag missing (dev only) |
| TLS | N/A | ✓ | Handled by reverse proxy / load balancer |

---

### Compliance Mapping (if applicable)

| Finding | Regulation | Control | Status |
|---|---|---|---|
| PII in logs | GDPR | Art. 32 — encryption of personal data | Violation |
| No audit log | HIPAA | §164.312(b) — audit controls | Violation |
| Hardcoded secret | SOC 2 | CC6.1 — logical access security | Violation |

---

### Secrets Scan

| Type | Finding | Location | Status |
|---|---|---|---|
| API key in source | src/config.ts:15 | Current + git history | CRITICAL — rotate + remove |
| .env in .gitignore | .gitignore | ✓ | OK |

---

### Summary

- **CRITICAL:** N findings
- **HIGH:** N findings
- **MEDIUM:** N findings
- **LOW:** N findings
- **CVEs:** N vulnerable, N not affected, N patched
- **Best practices gaps:** N
- **Compliance violations:** N (if applicable)

---
*Audit by security-audit skill — [timestamp]*
