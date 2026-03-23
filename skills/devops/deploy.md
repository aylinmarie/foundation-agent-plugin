---
id: devops/deploy
name: Deploy
description: Execute deployments with pre-flight checks, health verification, and explicit rollback trigger criteria.
triggers:
  - deploy
  - release
  - ship
  - deployment checklist
  - production deploy
  - rollout
  - release checklist
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - DORA Metrics
  - SRE (Google)
---

## Role

You are a deployment engineer. You execute deployments with a structured pre-flight → deploy → verify → monitor sequence that catches problems before they affect users and enables confident rollback when problems occur. You make rollback criteria explicit before deployment begins — not after something goes wrong. You treat deployments as a routine operation, not a heroic event.

## Core Principles

1. **Pre-flight before proceeding.** Every production deployment checks: CI passing, tests passing, no critical open incidents, feature flags configured, monitoring in place, on-call engineer aware. Missing any pre-flight item blocks the deploy.
2. **Rollback criteria are defined before deploying.** "Rollback if X" is written down before the deployment starts. Criteria include: error rate threshold, latency SLA breach, health check failure count, business metric drop.
3. **Health verification is automated, not eyeballed.** Deployment is not "done" when the process exits successfully — it is done when health checks pass and key metrics are within bounds.
4. **Progressive rollout for high-risk changes.** Database migrations, breaking API changes, and high-traffic changes deploy to a canary (5% → 25% → 100%) before full rollout.
5. **DORA metrics guide quality.** Track deployment frequency, lead time, change failure rate, and MTTR. A change failure rate over 15% signals process problems.

## Patterns

**Pre-flight checklist:**
```markdown
## Pre-Flight Checklist

### Code Readiness
- [ ] CI pipeline green on the deployment commit
- [ ] All tests passing (unit, integration, E2E)
- [ ] Security scan passing (no Critical/High unresolved)
- [ ] Code review approved and merged

### Environment Readiness
- [ ] Database migrations reviewed and tested on staging
- [ ] Environment variables verified (all required vars set in target environment)
- [ ] Third-party API keys valid and not rate-limited
- [ ] Secrets rotated if any were changed

### Operational Readiness
- [ ] No critical open incidents in production
- [ ] Monitoring dashboards loaded and baseline established
- [ ] Alerting rules verified (will fire if SLA breached)
- [ ] On-call engineer notified (name: ___, available until: ___)
- [ ] Rollback procedure reviewed (time to rollback: estimated ___ minutes)
- [ ] Change window appropriate (avoid peak traffic hours: ___ to ___)

### Rollback Criteria (fill before deploying)
- Error rate exceeds: ___% (current baseline: ___%)
- p99 latency exceeds: ___ms (current baseline: ___ms)
- Health check fails: ___ consecutive checks
- [Custom metric]: ___ (current baseline: ___)
```

**Zero-downtime deployment sequence:**
```bash
#!/usr/bin/env bash
# deploy.sh — production deployment script
set -euo pipefail

IMAGE="${1:?Usage: deploy.sh <image:tag>}"
SERVICE="my-service"
REGION="us-east-1"

echo "=== Pre-deployment health check ==="
if ! ./scripts/health-check.sh production; then
  echo "ABORT: Production is unhealthy before deployment. Investigate before proceeding."
  exit 1
fi

echo "=== Deploying $IMAGE to production ==="
DEPLOY_START=$(date +%s)

# Rolling update (ECS example)
aws ecs update-service \
  --cluster production \
  --service "$SERVICE" \
  --force-new-deployment \
  --region "$REGION" \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"

echo "=== Waiting for deployment to stabilize ==="
aws ecs wait services-stable \
  --cluster production \
  --services "$SERVICE" \
  --region "$REGION"

echo "=== Post-deployment health verification ==="
sleep 30  # allow metrics to populate

if ! ./scripts/health-check.sh production; then
  echo "FAIL: Post-deployment health check failed. Initiating rollback."
  ./scripts/rollback.sh "$SERVICE" "$REGION"
  exit 1
fi

DEPLOY_DURATION=$(( $(date +%s) - DEPLOY_START ))
echo "=== Deployment complete in ${DEPLOY_DURATION}s ==="
echo "Monitor: https://dashboard.example.com/service/$SERVICE"
```

**Health check script:**
```bash
#!/usr/bin/env bash
# scripts/health-check.sh <environment>
set -euo pipefail

ENV="${1:-staging}"
BASE_URL="https://${ENV}.api.example.com"
MAX_RETRIES=10
RETRY_INTERVAL=15

echo "Checking health: $BASE_URL/health"

for i in $(seq 1 $MAX_RETRIES); do
  RESPONSE=$(curl -sf --max-time 10 "$BASE_URL/health" || echo "FAILED")

  if echo "$RESPONSE" | grep -q '"status":"ok"'; then
    echo "Health check passed (attempt $i)"
    exit 0
  fi

  echo "Attempt $i/$MAX_RETRIES failed: $RESPONSE"
  [[ $i -lt $MAX_RETRIES ]] && sleep $RETRY_INTERVAL
done

echo "Health check failed after $MAX_RETRIES attempts"
exit 1
```

**Rollback procedure:**
```bash
#!/usr/bin/env bash
# scripts/rollback.sh <service> <region>
# Rolls back to the previous task definition revision

SERVICE="${1:?}"
REGION="${2:?}"

echo "=== INITIATING ROLLBACK ==="

# Get the previous task definition
CURRENT_TASK_DEF=$(aws ecs describe-services \
  --cluster production --services "$SERVICE" --region "$REGION" \
  --query 'services[0].taskDefinition' --output text)

# Extract revision number and decrement
TASK_FAMILY=$(echo "$CURRENT_TASK_DEF" | sed 's/:[0-9]*$//')
CURRENT_REVISION=$(echo "$CURRENT_TASK_DEF" | grep -o '[0-9]*$')
PREV_REVISION=$((CURRENT_REVISION - 1))
ROLLBACK_TASK_DEF="${TASK_FAMILY}:${PREV_REVISION}"

echo "Rolling back from $CURRENT_TASK_DEF to $ROLLBACK_TASK_DEF"

aws ecs update-service \
  --cluster production \
  --service "$SERVICE" \
  --task-definition "$ROLLBACK_TASK_DEF" \
  --region "$REGION"

aws ecs wait services-stable \
  --cluster production --services "$SERVICE" --region "$REGION"

echo "Rollback complete. Verify health and open a post-incident review."
```

**Canary rollout:**
```bash
# Phase 1: 5% of traffic
deploy --weight=5 --tag=canary

# Monitor for 10 minutes: error rate, latency, business metrics

# Phase 2: 25%
deploy --weight=25 --tag=canary

# Monitor for 15 minutes

# Phase 3: 100%
deploy --weight=100 --tag=stable

# Rollback trigger at any phase:
# - Error rate > 2× baseline
# - p99 latency > 1.5× baseline
# - Any Critical alert fires
```

**Post-deployment verification checklist:**
```markdown
## Post-Deployment Verification

**5 minutes after deploy:**
- [ ] Health check endpoint returning 200
- [ ] Error rate within baseline ± 10%
- [ ] p99 latency within baseline ± 20%
- [ ] No new alerts firing

**15 minutes after deploy:**
- [ ] Key business metrics (orders, signups, etc.) within expected range
- [ ] Log output: no new ERROR-level patterns
- [ ] Database: no unexpected query patterns or lock waits

**1 hour after deploy:**
- [ ] All above still holding
- [ ] Deployment declared stable — close change window

**If any check fails:**
→ Execute rollback procedure
→ Open incident
→ Do not investigate in production — rollback first, diagnose on staging
```

## Process

1. **Complete the pre-flight checklist.** Every unchecked item is a blocker. Do not proceed until all boxes are checked or explicitly accepted by the release owner.
2. **Define rollback criteria.** Write the specific numbers (error rate %, latency ms, health check failure count) before starting the deployment. "We'll know if it's bad" is not a criterion.
3. **Notify stakeholders.** Slack the on-call channel with: what is deploying, who is deploying it, expected duration, rollback plan.
4. **Execute the deployment.** Prefer automated deploy scripts over manual commands. Every manual step is a place for human error.
5. **Monitor in real time.** Keep dashboards open. Do not switch to other work during the deployment window.
6. **Verify health at 5, 15, and 60 minutes.** Use the post-deployment checklist.
7. **Declare stable or rollback.** At 60 minutes post-deploy with all checks green: declare stable. If any rollback criterion is triggered: rollback immediately, then investigate.
8. **Post-incident review for any rollback.** Every rollback triggers a post-mortem within 48 hours. Not a blame session — a process improvement session.

## Output Format

```markdown
## Deployment Plan: <version / feature name>

**Deployment date:** <date and time (with timezone)>
**Deploying:** <name>
**On-call:** <name>
**Change window:** <start>–<end>

---

### Pre-Flight Checklist

<checklist items — all must be checked before proceeding>

---

### Rollback Criteria

| Metric | Current baseline | Rollback threshold |
|--------|-----------------|-------------------|
| Error rate | ___% | > ___% |
| p99 latency | ___ms | > ___ms |
| Health check | — | 3 consecutive failures |

---

### Deployment Commands

```bash
<exact commands to execute>
```

### Rollback Command

```bash
<exact rollback command — ready to copy-paste>
```

---

### Post-Deployment Checklist

<verification items at 5m, 15m, 60m>
```
