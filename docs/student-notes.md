# Student Audit Notes

Use this file to document your governance audit findings.

## Task 1: Audit Findings

### Weakness 1: Deployment triggers on any branch

**Location (file + line number or GitHub setting path):**
```
.github/workflows/deploy.yml, lines 3-6
```

**Evidence:**
```
on:
  push:
    branches:
      - '*'
```

**Risk:**
- Any push to any branch (feature branch, throwaway branch, typo branch) runs the deploy job straight to production.
- Untested/incomplete code can reach production without ever touching `main`.
- No human gets a chance to say no before it ships.

**Root Cause:**
- `branches: ['*']` was left as a wildcard from early scaffolding and never tightened before the workflow went live.

**Recommended Fix:**
- Restrict trigger to `branches: [main]` (or `release/*`), and additionally gate the job with `environment: production` so branch restriction is enforced twice — once at trigger, once at environment level.

---

### Weakness 2: No `environment:` field on the deploy job

**Location:**
```
.github/workflows/deploy.yml, deploy job (lines 10-15)
```

**Evidence:**
```
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
```
No `environment:` key anywhere in the job.

**Risk:**
- Without an `environment:` reference, GitHub has nothing to attach required reviewers, wait timers, or deployment-branch rules to — the job can never be gated, no matter what's configured in Settings → Environments.
- This is the root enabler for weaknesses 3, 5, 6, 7 below.

**Root Cause:**
- Workflow was written to just "get deploys working" with no environment protection designed in from the start.

**Recommended Fix:**
- Add `environment: production` to the `deploy` job. That single line is what makes GitHub actually enforce whatever protection rules get configured on the `production` environment.

---

### Weakness 3: Production secrets stored at repository scope

**Location:**
```
.github/workflows/deploy.yml, lines 26-29; Settings → Secrets and variables → Actions (repository secrets)
```

**Evidence:**
```
env:
  DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_TOKEN: ${{ secrets.API_TOKEN }}
```

**Risk:**
- Repo-level secrets are readable by any workflow run in the repo, triggered from any branch — including a feature branch someone pushed by mistake.
- Because there's no environment protection (weakness 2), these production credentials are exposed to exactly the "any branch deploys" problem in weakness 1.

**Root Cause:**
- Secrets were added at the repo level because no `production` environment existed yet to scope them to.

**Recommended Fix:**
- Move `DEPLOY_KEY`, `DATABASE_URL`, `API_TOKEN` into Settings → Environments → `production` → Environment secrets, and delete the repo-level copies. Only jobs with `environment: production` can then read them.

---

### Weakness 4: No required reviewers on production deploys

**Location:**
```
Settings → Environments (no `production` environment exists yet)
```

**Evidence:**
```
Environments list is empty — nothing to attach "Required reviewers" to.
```

**Risk:**
- Deployment happens the instant CI/the trigger fires — no person signs off. A compromised token, bad merge, or accidental push ships straight to prod.

**Root Cause:**
- Same root cause as weakness 2 — no environment object exists to hold this rule.

**Recommended Fix:**
- Create `production` environment, enable "Required reviewers", add at least 1-2 reviewers (team members with deploy authority).

---

### Weakness 5: No deployment branch restriction at the environment level

**Location:**
```
Settings → Environments → production (does not exist yet)
```

**Evidence:**
```
No environment, therefore no "Deployment branches" rule.
```

**Risk:**
- Even if the workflow trigger (weakness 1) is fixed, without an environment-level branch rule there's no second layer of enforcement — a workflow file change or `workflow_dispatch` from a non-main branch could still deploy.

**Root Cause:**
- No environment configured.

**Recommended Fix:**
- On the `production` environment, enable "Deployment branches and tags" → restrict to `main` only.

---

### Weakness 6: No wait timer before deployment executes

**Location:**
```
Settings → Environments → production (does not exist yet)
```

**Evidence:**
```
No environment, therefore no wait timer configured.
```

**Risk:**
- Even after a reviewer approves, deployment fires instantly — no grace period to catch an approval made in error or to abort if something looks wrong immediately after approval.

**Root Cause:**
- No environment configured.

**Recommended Fix:**
- Add a wait timer (1 minute for this lab; 60+ seconds typical in real production) on the `production` environment.

---

### Weakness 7: Broad/unscoped job permissions

**Location:**
```
.github/workflows/deploy.yml, lines 12-14
```

**Evidence:**
```
permissions:
  contents: read
  id-token: write
```

**Risk:**
- `id-token: write` grants OIDC token-minting capability workflow-wide, even though nothing in the deploy steps uses it (no `aws-actions/configure-aws-credentials` or similar OIDC exchange is present). An unnecessary permission is attack surface for token theft/misuse if any step is compromised.

**Root Cause:**
- Permission was probably copy-pasted from a cloud-deploy template without checking whether it's actually used.

**Recommended Fix:**
- Drop `id-token: write` unless/until an OIDC-based cloud deploy step is actually added. Keep permissions to the minimum the job's steps require (principle of least privilege).

---

## Task 2: Configuration Actions

Record what you configured:

- [ ] Created production environment in GitHub
- [ ] Added branch protection: only `main` can deploy
- [ ] Added required reviewers: ___ (number)
- [ ] Added wait timer: ___ seconds
- [ ] Created environment secrets:
  - [ ] DEPLOY_KEY
  - [ ] DATABASE_URL
  - [ ] API_TOKEN
- [ ] Updated `.github/workflows/deploy.yml` with `environment: production`

### Configuration Checklist

**GitHub Settings → Environments → production:**

- [ ] Environment name is "production"
- [ ] Branch protection is enabled
- [ ] Only "main" branch is allowed
- [ ] Required reviewers: minimum 2
- [ ] Wait timer: 60 seconds
- [ ] Environment secrets are configured

**Workflow file `.github/workflows/deploy.yml`:**

- [ ] Job includes `environment: production`
- [ ] Deployment only triggers on `push: branches: [main]`
- [ ] Secrets are referenced correctly

---

## Task 3: Validation Evidence

### Deployment Trigger

**How I triggered the deployment:**
```
[Describe what change you made and how you pushed it]
```

**Workflow Run ID:**
```
[Paste the workflow run ID from GitHub Actions]
```

### Approval Step Captured

**Screenshot or description of workflow pause:**
```
[Describe what you saw when the workflow paused at the approval step]
```

**Reviewers shown:**
- Reviewer 1: _______________
- Reviewer 2: _______________

**Approval timestamp:** _______________

### Deployment Completion

**Wait timer duration:** ___ seconds

**Workflow completed successfully:**  ☐ Yes  ☐ No

**Final deployment status:**
```
[Copy the final job status from GitHub Actions logs]
```

---

## Summary

### Total Weaknesses Found

- [ ] 1 weakness
- [ ] 2 weaknesses
- [ ] 3 weaknesses
- [ ] 4 weaknesses
- [ ] 5 weaknesses
- [ ] 6 weaknesses
- [ ] 7+ weaknesses

### Configuration Completion

- [ ] 0–25% complete
- [ ] 25–50% complete
- [ ] 50–75% complete
- [ ] 75–100% complete

### Validation Status

- [ ] Workflow pauses at approval ✓
- [ ] Reviewers notified ✓
- [ ] Wait timer enforced ✓
- [ ] Deployment proceeds after approval ✓

---

## Reflection

**What was the most critical weakness you found?**

```

```

**Which fix was most important to implement?**

```

```

**What surprised you about deployment governance?**

```

```

**How would you apply this to a real production system?**

```

```

---

## Questions for Your Instructor

List any questions that came up during the lab:

1. 

2. 

3. 

