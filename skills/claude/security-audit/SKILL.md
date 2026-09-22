---
name: security-audit
description: "Contextual security audit: traces features, fetches CVEs, best practices."
---
# Security Audit

Contextual security audit that traces from features to implementation, fetches
recent CVEs for the exact dependency stack, retrieves technology-specific best
practices, and verifies the code covers them. Not a pattern-grep scanner — it
understands what each feature does and audits it against the real threat model.

**Not the same as `repo-audit`**, which does pattern-grep security scanning.
This skill maps the attack surface, traces which PR implemented which feature,
fetches live vulnerability data, and compares implementation against best
practices for the specific technology.

## When to Use

- User asks for a security audit of a repo or feature
- User asks to check for vulnerabilities in their code
- User asks "is this code secure?"
- After a major feature implementation, before shipping
- Periodic security review (monthly/quarterly)
- Before compliance review or penetration testing

## Don't Use For

- Single diff/PR review → use `code-review` (lane 4 is security)
- General code quality → use `repo-health`
- Bug finding across the repo → use `repo-audit`

## Prerequisites

- A git repository with code to audit
- `WebSearch` for CVE and best practice retrieval
- `gh` CLI for PR history and GitHub Security Advisories
- Optional: Graphify MCP tools for PR impact analysis
- Optional: `gitleaks`, `trufflehog`, `npm audit`, `pip-audit`, `cargo audit`, `govulncheck`
- Linear MCP tools for ticket creation (Phase 6)

## Procedure

### Phase 1 — Technology Fingerprint

Identify the exact stack so vulnerability searches are precise.

1. Detect the technology stack:
   ```bash
   # Package manifests
   cat package.json go.mod Cargo.toml pyproject.toml requirements.txt pom.xml build.gradle Gemfile composer.json mix.exs pubspec.yaml 2>/dev/null
   # Framework configs
   cat next.config.* nuxt.config.* vite.config.* django settings flask app fastapi main 2>/dev/null
   # Infrastructure
   cat Dockerfile docker-compose*.yml terraform/*.tf apprunner.yaml serverless.yml 2>/dev/null
   ```

2. Extract exact dependency names and versions from lockfiles:
   ```bash
   # Node
   cat package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null | grep -E '"[a-z@/-]+":' | head -50
   # Python
   pip freeze 2>/dev/null || cat requirements*.txt 2>/dev/null
   # Go
   cat go.sum 2>/dev/null | awk '{print $1, $2}' | head -50
   # Rust
   cat Cargo.lock 2>/dev/null | grep -A1 'name = ' | head -50
   ```

3. Record under `## Technology Fingerprint` with exact versions.

### Phase 2 — Attack Surface Mapping

Identify every entry point where untrusted data enters the system. This is where
attacks land.

4. **API endpoints** — grep for route definitions:
   ```bash
   # Express/Koa/Fastify
   grep -rn "app\.(get\|post\|put\|delete\|patch)\|router\.(get\|post\|put\|delete\|patch)" --include="*.ts" --include="*.js" src/
   # FastAPI / Flask / Django
   grep -rn "@app\.(get\|post\|put\|delete)\|@router\.(get\|post\|put\|delete)\|path(" --include="*.py" .
   # Go
   grep -rn "http\.HandleFunc\|r\.(GET\|POST\|PUT\|DELETE)\(" --include="*.go" .
   # GDScript (Godot)
   grep -rn "func _on_\|signal \|rpc(" --include="*.gd" .
   ```

5. **Webhooks** — grep for webhook handlers:
   ```bash
   grep -rn "webhook" --include="*.ts" --include="*.py" --include="*.go" --include="*.js" -i .
   ```

6. **File uploads** — grep for file handling:
   ```bash
   grep -rn "multer\|upload\|FormData\|multipart\|upload_file\|request\.files\|FileField" --include="*.ts" --include="*.py" --include="*.js" --include="*.go" -i .
   ```

7. **External API calls** — grep for outbound HTTP:
   ```bash
   grep -rn "fetch(\|axios\|requests\.(get\|post)\|http\.Get\|http\.Post\|HttpClient" --include="*.ts" --include="*.py" --include="*.go" --include="*.js" .
   ```

8. **User input** — grep for input parsing:
   ```bash
   grep -rn "req\.(body\|query\|params\|headers)\|request\.(POST\|GET\|data\|headers\|args)\|ctx\.(body\|query\|params)\|request\.form" --include="*.ts" --include="*.py" --include="*.js" --include="*.go" .
   ```

9. For each entry point, record: route/method, data source, auth required, what
   it does. Record under `## Attack Surface Map`.

### Phase 3 — Feature and PR Tracing

Trace which PRs implemented which security-relevant features, then audit each.

10. **Identify security-relevant features** — from the attack surface map, these
    are: authentication, authorization, password handling, session management,
    file upload, payment processing, data access, external API calls, admin
    endpoints.

11. **Trace implementation via git history**:
    ```bash
    # Find PRs that touched auth/security code
    git log --oneline --all -- src/auth/ src/security/ src/middleware/ "src/**/auth*" "src/**/security*"
    # Find when each security feature was introduced
    git log --diff-filter=A --oneline -- "src/**/auth*" "src/**/security*" "src/**/middleware*"
    # Get the full diff of each security PR
    gh pr list --state merged --search "auth OR security OR login OR password OR jwt" --limit 20
    ```

12. **Optional: Use Graphify for impact analysis** — if Graphify MCP is available:
    - `mcp__graphify__get_pr_impact` — see which nodes a PR touches
    - `mcp__graphify__triage_prs` — find PRs that overlap in security-critical areas
    - `mcp__graphify__query_graph` — "what connects auth to the database?"

13. **Read the actual implementation** — for each security-relevant feature, use
    `Read` and `Grep` to read the full implementation. Understand:
    - What it does
    - What input it accepts
    - What output it produces
    - What it trusts (and shouldn't)
    - What it doesn't validate

### Phase 4 — Main Security Concerns Audit

For each security-relevant area, audit against the standard checklist:

14. **Authentication & Authorization:**
    - Passwords hashed with bcrypt/argon2 (not MD5/SHA1)
    - JWT signed with strong secret, short expiry, refresh token rotation
    - Session tokens: httpOnly, secure, sameSite cookies
    - RBAC/ABAC enforced on every privileged endpoint
    - No IDOR (Insecure Direct Object Reference) — check object ownership
    - Rate limiting on auth endpoints (login, signup, password reset)

15. **Injection:**
    - SQL: parameterized queries or ORM, no string concatenation
    - NoSQL: sanitized queries, no raw $where
    - Command: no os.system/subprocess with shell=True + user input
    - LDAP: parameterized LDAP queries
    - Template: auto-escaping enabled, no raw HTML output with user input

16. **XSS (Cross-Site Scripting):**
    - innerHTML/textContent used correctly (no innerHTML with user data)
    - React/Vue auto-escaping relied on, no dangerouslySetInnerHTML
    - CSP (Content Security Policy) header set
    - Input sanitized before display

17. **CSRF (Cross-Site Request Forgery):**
    - CSRF tokens on state-changing operations
    - SameSite cookie attribute set
    - Origin/Referer header validated

18. **File Upload:**
    - File type validation (MIME + extension, not just extension)
    - File size limits
    - Files stored outside web root
    - Filename sanitized (no path traversal)
    - Files scanned for malicious content if user-facing

19. **Data Protection:**
    - PII not logged
    - Encryption at rest (DB, file storage)
    - Encryption in transit (TLS everywhere)
    - Secrets not hardcoded (scan git history)
    - Data retention policy enforced

20. **Configuration Security:**
    - CORS configured (not wildcard with credentials)
    - Helmet/secure headers enabled
    - Debug mode off in production
    - Error messages don't leak stack traces to users
    - Environment variables not exposed to client
    - Cookie flags: httpOnly, secure, sameSite

21. **Dependency Security:**
    - Versions pinned (no floating tags in production)
    - Lockfiles committed
    - No typosquatting risk (check for similar-named packages)
    - Dependency confusion risk assessed (private vs public registry)

### Phase 5 — Vulnerability Research (Live)

Fetch recent vulnerabilities for the exact dependency stack from Phase 1.

22. **Run local dependency scanners:**
    ```bash
    # Node
    npm audit --json 2>/dev/null || pnpm audit --json 2>/dev/null || yarn audit --json 2>/dev/null
    # Python
    pip-audit --format json 2>/dev/null || safety check --json 2>/dev/null
    # Go
    govulncheck ./... 2>&1
    # Rust
    cargo audit --json 2>/dev/null
    ```

23. **Check GitHub Security Advisories:**
    ```bash
    gh api repos/{owner}/{repo}/dependabot/alerts --jq '.[] | select(.state == "open") | {severity, advisory: .security_advisory.summary, package: .security_vulnerability.package.name, vulnerable_version: .security_vulnerability.vulnerable_version_range}' 2>/dev/null
    ```

24. **Web search for recent CVEs** — for each major dependency, search for
    recent vulnerabilities:
    Use `WebSearch` with queries like:
    - "[dependency name] [version] CVE 2026"
    - "[dependency name] vulnerability 2026"
    - "[framework name] security advisory 2026"

25. **Check exploit availability** — for each CVE found:
    - Search ExploitDB: `WebSearch` "[CVE-ID] exploit"
    - Search GitHub: `WebSearch` "[CVE-ID] PoC site:github.com"
    - Public exploit available → CRITICAL
    - No public exploit but patch available → HIGH
    - No public exploit, no patch → HIGH (mitigation needed)

26. **Check OSV database** — search for each dependency:
    Use `WebSearch` with queries like:
    - "[dependency name] OSV vulnerability"
    - "osv.dev [dependency name]"

27. For each vulnerability found, record:
    - CVE ID, dependency, installed version, vulnerable range
    - Whether the code uses the vulnerable function/pattern
    - Whether an exploit is public
    - Fix: upgrade version, patch, or workaround
    - Status: VULNERABLE / NOT AFFECTED / PATCHED

### Phase 6 — Best Practices Retrieval

For each technology in the stack, fetch current security best practices and
compare against the implementation.

28. **Web search for best practices** — use `WebSearch` for each:
    - "[framework] security best practices 2026"
    - "[technology] security checklist"
    - "[auth method] security best practices"
    - "[database] security hardening"
    - "OWASP [framework] security guide"

29. **Framework built-in security features** — many frameworks have security
    features not enabled by default. Check which are enabled:
    - CSRF middleware: installed? enabled on all routes?
    - Rate limiting: installed? configured? on which routes?
    - Helmet/secure headers: installed? configured?
    - CORS: configured? not wildcard?
    - Input validation: schema validation (zod, joi, pydantic, class-validator)?
    - Parameterized queries: ORM used? or raw queries?

30. **Compare implementation against best practices:**
    - For each best practice, check: is it applicable? is it implemented?
    - Record as: ✓ Implemented / ✗ Missing / ⚠ Partial / N/A
    - For each gap, provide the specific recommendation

31. **OWASP mapping** — map each finding to:
    - OWASP Top 10 2021 category (A01-A10)
    - CWE ID (Common Weakness Enumeration)
    - This makes the report actionable for compliance

### Phase 7 — Secrets and Configuration Scan

32. **Secrets scan (current + history):**
    ```bash
    # Current state
    grep -rn "(api_key|secret|password|token|passwd)\s*=\s*['\"][^'\"]{6,}['\"]" --include="*.ts" --include="*.py" --include="*.go" --include="*.js" --include="*.env*" .
    # Git history
    gitleaks detect --source . 2>/dev/null || trufflehog git file://. 2>/dev/null
    # Or without tools:
    git log -p --all 2>/dev/null | grep -iE "(api_key|secret|password|token|passwd)\s*=" | head -20
    ```

33. **Configuration security:**
    ```bash
    # CORS
    grep -rn "cors\|Access-Control-Allow-Origin" --include="*.ts" --include="*.py" --include="*.js" .
    # Helmet / secure headers
    grep -rn "helmet\|X-Frame-Options\|X-Content-Type-Options\|Strict-Transport-Security" --include="*.ts" --include="*.py" --include="*.js" .
    # Debug mode
    grep -rn "DEBUG\|debug\s*=" --include="*.ts" --include="*.py" --include="*.js" . | grep -v node_modules
    # Cookie flags
    grep -rn "httpOnly\|secure:\|sameSite" --include="*.ts" --include="*.py" --include="*.js" .
    # TLS config
    grep -rn "tls\|ssl\|https" --include="*.ts" --include="*.py" --include="*.js" --include="*.go" . | grep -v node_modules
    ```

### Phase 8 — Report and Remediation

34. **Assign CVSS scores** — for each finding, estimate the CVSS 3.1 score:
    - Use the CVSS calculator mental model: Attack Vector, Attack Complexity,
      Privileges Required, User Interaction, Scope, Confidentiality/Integrity/Availability impact
    - Map to severity: 9.0-10.0 CRITICAL, 7.0-8.9 HIGH, 4.0-6.9 MEDIUM, 0.1-3.9 LOW

35. **Write fix suggestions with code** — for each finding:
    - State the vulnerability
    - Show the vulnerable code (file:line)
    - Show the fix (actual code, not "use parameterized queries")
    - Reference the best practice or CVE that drove the recommendation

36. **Compliance mapping** (if applicable):
    - GDPR: Art. 32 (encryption), Art. 25 (data protection by design)
    - HIPAA: §164.312 (access control, encryption, audit controls)
    - SOC 2: CC6.1 (logical access), CC7.1 (system monitoring)
    - PCI-DSS: Req 6 (secure coding), Req 8 (authentication)
    - Map each finding to the specific control it violates

37. **Write the report** to `.security-audit/REPORT.md` using
    `templates/report.md`.

38. **Present in chat:** the attack surface map, top 5 CRITICAL/HIGH findings, the
    CVE summary, and the report path. Do not paste the full report.

### Phase 9 — Create Linear Tickets (optional, if user approves)

39. Resolve the team: `mcp__linear__get_teams`
40. For each CRITICAL and HIGH finding:
    - Create a Linear issue:
      - **Title:** `security: [CWE/OWASP] [finding]` (e.g., "security: CWE-89 SQL injection in search endpoint")
      - **Description:** the finding, file:line, CVSS score, vulnerable code, fix
        suggestion with code, CVE reference (if applicable)
      - **Priority:** 0 (urgent) for CRITICAL, 1 (high) for HIGH
      - **Label:** "security" if available
41. Group related findings into a parent umbrella ticket
42. Confirm ticket URLs back to the user

## Severity Levels

| Level | CVSS | Meaning | Action |
|---|---|---|---|
| CRITICAL | 9.0-10.0 | Exploitable vulnerability with public exploit, or data breach risk | Linear ticket, fix immediately |
| HIGH | 7.0-8.9 | Vulnerability with no public exploit, or missing critical security control | Linear ticket, fix next sprint |
| MEDIUM | 4.0-6.9 | Best practice gap, hardening needed | Linear ticket, backlog |
| LOW | 0.1-3.9 | Information disclosure, minor config | List in report |

## Pitfalls

- **Scanning without understanding the code** — this is not a grep scanner. Read the actual implementation of each security-relevant feature before reporting.
- **Stale vulnerability data** — CVEs are published daily. Always web search for recent ones, don't rely only on local scanners.
- **False positives from dependency scanners** — `npm audit` reports vulnerabilities in packages you don't import transitively. Verify the vulnerable function is actually used before reporting.
- **Missing the git history** — secrets committed and removed are still in history. Scan history, not just current state.
- **Reporting on code you haven't read** — "this function might be vulnerable" is not a finding. Read it, confirm it, then report with file:line.
- **Best practices without version context** — "use CSRF protection" is generic. "Express 4.x supports csurf middleware, which is not installed" is actionable.
- **Not checking exploit availability** — a CVE with a public PoC is CRITICAL; the same CVE without one is HIGH. The distinction matters.
- **Ignoring the attack surface** — auditing code without knowing where untrusted data enters leads to findings on unreachable code. Map the surface first.
- **Over-reporting LOW findings** — flooding the report with minor config issues buries the CRITICAL ones. Keep LOW findings in the report but don't create Linear tickets for them.
- **Not verifying framework built-in features** — many frameworks have security features that are installed but not enabled, or enabled but misconfigured. Check configuration, not just package presence.

## Verification

- [ ] Technology fingerprint complete with exact versions
- [ ] Attack surface map covers all entry points
- [ ] Security-relevant features traced to PRs and implementation read
- [ ] All 7 main security concerns audited (auth, injection, XSS, CSRF, file upload, data protection, config)
- [ ] Local dependency scanners run (or noted as unavailable)
- [ ] GitHub Security Advisories checked
- [ ] Web search for recent CVEs completed for major dependencies
- [ ] Exploit availability checked for each CVE
- [ ] Best practices retrieved and compared for each technology
- [ ] Framework built-in security features checked (enabled + configured)
- [ ] OWASP + CWE mapping complete for each finding
- [ ] CVSS scores assigned
- [ ] Fix suggestions include actual code, not just recommendations
- [ ] Secrets scan includes git history
- [ ] Compliance mapping done (if applicable)
- [ ] Report written to `.security-audit/REPORT.md`
- [ ] Linear tickets created for CRITICAL/HIGH findings (if user approved)
