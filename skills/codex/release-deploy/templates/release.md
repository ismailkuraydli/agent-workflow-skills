## Release: v[VERSION] — [Date]

### Release Summary
- **Previous version:** v[OLD_VERSION]
- **New version:** v[NEW_VERSION]
- **Bump type:** [major/minor/patch/none]
- **PRs shipped:** N
- **Deploy status:** [success/failed/rolled back]

### Release Notes

#### Features
- [PR #N]: [description]

#### Bug Fixes
- [PR #N]: [description]

#### Security
- [PR #N]: [description]

#### Performance
- [PR #N]: [description]

#### Maintenance
- [PR #N]: [description]

### Pre-Deploy Checklist
- [ ] All PRs merged to default branch
- [ ] Version bumped in all files
- [ ] CHANGELOG.md updated
- [ ] CI green on default branch
- [ ] No open security alerts
- [ ] Database migrations ready and reversible
- [ ] Feature flags configured
- [ ] Environment variables verified
- [ ] Deploy target healthy

### Deploy
- **Strategy:** [rolling/blue-green/canary/recreate/feature-flag]
- **Mechanism:** [CI workflow / make deploy / Docker push / cloud CLI]
- **Start time:** [timestamp]
- **End time:** [timestamp]
- **Result:** [success/failed]

### Smoke Tests
| Check | URL | Expected | Actual | Status |
|---|---|---|---|---|
| Health | /health | 200 OK | 200 OK | ✓ |
| Auth | /auth/login | 200 + token | 200 + token | ✓ |
| Core feature | /api/[main] | 200 + data | 200 + data | ✓ |

### Rollback Plan
- **Method:** [git revert / kubectl rollout undo / heroku releases:rollback / etc.]
- **Previous version:** v[OLD_VERSION]
- **Database rollback:** [N/A / migration revert tested / not reversible — fix-forward]
- **Status:** [not needed / executed successfully / executed with issues]

### Post-Release
- **Linear tickets closed:** N
- **Issues encountered:** [none / list]
- **Monitoring window:** [15 min post-deploy — all clear / issues found]

---
*Released by release-deploy skill — [timestamp]*
