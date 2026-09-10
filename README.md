# DORA Metrics: Practical Implementation Guide

> How to measure Deployment Frequency, Lead Time, Change Failure Rate, and MTTR across fragmented toolchains — and the top tools that do it out of the box.

---

## Why DORA Metrics Matter

DORA (DevOps Research and Assessment) metrics are the industry standard for measuring software delivery performance. The [Accelerate State of DevOps Report](https://dora.dev) — backed by 10+ years of research across 39,000+ professionals — consistently shows that **elite performers deploy 200x more frequently with 2,555x faster lead times** than low performers.

These aren't vanity metrics. Teams that improve on DORA metrics see direct impact on:
- **Business outcomes**: Revenue, market share, customer satisfaction
- **Operational health**: Less burnout, lower unplanned work (22% less for elite teams)
- **Organizational performance**: 2x likelihood of exceeding goals

---

## The Four Metrics

| Metric | What It Measures | Elite | High | Medium | Low |
|--------|-----------------|-------|------|--------|-----|
| **Deployment Frequency** | How often you deploy to production | Many per day | Daily to weekly | Weekly to monthly | Monthly+ |
| **Lead Time for Changes** | Commit to running in production | < 1 day | 1 day - 1 week | 1 week - 1 month | > 1 month |
| **Change Failure Rate** | % of deployments causing incidents | 5% | 10% | 15% | 64% |
| **Mean Time to Restore (MTTR)** | Incident detection to resolution | < 1 hour | < 1 day | 1 day - 1 week | 1 month+ |

---

## How Computation Actually Works (The Hard Part)

### The Core Challenge: Data Spread Across Systems

In most real-world setups, no single tool owns the full picture:

```
┌──────────┐    ┌──────────┐    ┌───────────────┐    ┌───────────┐    ┌────────────┐
│  GitHub   │───>│  GHA CI  │───>│ Artifact Reg  │───>│  ArgoCD   │───>│ Production │
│ (commit)  │    │ (build)  │    │ (image:SHA)   │    │ (deploy)  │    │ (running)  │
└─────┬─────┘    └─────┬────┘    └───────────────┘    └─────┬─────┘    └─────┬──────┘
      │                │                                     │               │
      │          Events / Webhooks / APIs                    │               │
      v                v                                     v               v
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    DORA Metrics Platform                                        │
│                                                                                 │
│  Correlation Key: COMMIT SHA flows through every system                        │
│  Joins with: Incident management (PagerDuty / Jira / Opsgenie)                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### The Commit SHA: Universal Correlation Key

The **commit SHA** is the thread that ties everything together:

1. **GitHub**: Commit `abc123` is created (timestamp = `T_commit`)
2. **GHA**: Build triggered, image tagged `myapp:abc123`, pushed to Artifact Registry
3. **ArgoCD/Harness**: Deploys image `myapp:abc123` to production (timestamp = `T_deploy`)
4. **PagerDuty/Jira**: Incident linked to deployment containing `abc123`

Every DORA metric is computed by joining data across these timestamps.

### Per-Metric Computation

#### 1. Deployment Frequency

```
Formula: count(production_deployments) / time_period

Data Source: CD tool (ArgoCD sync events, Harness pipeline executions)

ArgoCD:
  - Application sync events where target = production
  - kubectl get applications -n argocd -o json → .status.history[]

Harness:
  - Pipeline executions with environment = production
  - GET /pipeline/api/pipelines/execution/summary?module=cd

GHA (if deploying via GHA):
  - Workflow runs where workflow_name contains "deploy" AND conclusion = "success"
  - GET /repos/{owner}/{repo}/actions/runs?event=deployment
```

#### 2. Lead Time for Changes

```
Formula: T_deploy - T_commit (for each commit SHA in the deployment)

Data Sources: GitHub API (commit timestamp) + CD tool (deploy timestamp)

Step 1 — Get commit timestamp:
  GET /repos/{owner}/{repo}/commits/{sha}
  → .commit.author.date = T_commit

Step 2 — Get deploy timestamp:
  ArgoCD: Application sync history → .deployedAt for the revision containing SHA
  Harness: Pipeline execution → .startTs for the deployment containing SHA

Step 3 — Compute:
  lead_time = T_deploy - T_commit
  Report: median (p50) and p95 across all commits in the period
```

#### 3. Change Failure Rate

```
Formula: deployments_causing_incidents / total_deployments * 100

Data Sources: CD tool (deployment list) + Incident management (incidents)

Step 1 — List all production deployments in period (from CD tool)
Step 2 — List all incidents in period (from PagerDuty/Jira)
Step 3 — Link incidents to deployments:
  - Time correlation: incident opened within N hours of deployment
  - Explicit link: incident references commit SHA or deployment ID
  - Rollback detection: a revert/rollback deploy shortly after a forward deploy

The hard part: defining "caused by a deployment" vs coincidental timing.
Best practice: require explicit incident tagging during postmortem.
```

#### 4. Mean Time to Restore (MTTR)

```
Formula: avg(T_resolved - T_detected) for all incidents

Data Sources: Incident management system (primary)

PagerDuty:
  GET /incidents?statuses[]=resolved&since={start}&until={end}
  MTTR = resolved_at - created_at (per incident)

Jira:
  JQL: project = OPS AND type = Incident AND resolved >= -30d
  MTTR = resolutiondate - created (per issue)

Opsgenie:
  GET /v2/incidents?status=resolved
  MTTR = report.closedAt - report.detectedAt
```

---

## How Big Organizations Do This in Practice

### Pattern 1: Event-Driven Collection (Most Common)

Every tool in the pipeline emits events (webhooks) to a central collector:

```yaml
# GitHub webhook → on push/merge
{
  "event": "push",
  "sha": "abc123",
  "timestamp": "2026-09-10T10:00:00Z",
  "repository": "myapp",
  "branch": "main"
}

# GHA webhook → on workflow completion
{
  "event": "workflow_run",
  "sha": "abc123",
  "conclusion": "success",
  "workflow": "build-and-push",
  "completed_at": "2026-09-10T10:08:00Z"
}

# ArgoCD webhook/notification → on sync
{
  "event": "sync",
  "sha": "abc123",
  "application": "myapp-prod",
  "synced_at": "2026-09-10T10:12:00Z",
  "status": "Healthy"
}

# PagerDuty webhook → on incident
{
  "event": "incident.resolved",
  "service": "myapp",
  "created_at": "2026-09-10T11:00:00Z",
  "resolved_at": "2026-09-10T11:25:00Z"
}
```

### Pattern 2: API Polling (Simpler but Delayed)

A scheduled job (cron, GHA scheduled workflow) polls each tool's API periodically:

```yaml
# .github/workflows/dora-collector.yml
name: DORA Metrics Collection
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours

jobs:
  collect:
    runs-on: ubuntu-latest
    steps:
      - name: Collect GHA deploy runs
        run: |
          gh api repos/${{ github.repository }}/actions/runs \
            --jq '.workflow_runs[] | select(.name | contains("deploy")) | 
            {sha: .head_sha, status: .conclusion, completed: .updated_at}'

      - name: Collect ArgoCD syncs
        run: |
          argocd app list -o json | jq '.[] | 
            {app: .metadata.name, revision: .status.sync.revision, 
             health: .status.health.status, synced: .status.operationState.finishedAt}'

      - name: Collect PagerDuty incidents
        run: |
          curl -s -H "Authorization: Token token=$PD_TOKEN" \
            "https://api.pagerduty.com/incidents?since=$(date -d '-6 hours' -Iseconds)" \
            | jq '.incidents[] | {id: .id, service: .service.summary, 
              created: .created_at, resolved: .resolved_at}'

      - name: Compute and store metrics
        run: python compute_dora.py  # Join data, compute metrics, push to dashboard
```

### Pattern 3: Deploy Markers (Pragmatic Shortcut)

Instead of correlating across systems, instrument the deploy step itself:

```yaml
# In your ArgoCD post-sync hook or Harness pipeline
- name: Record deployment
  script: |
    curl -X POST https://your-dora-platform/api/deployments \
      -d '{
        "service": "myapp",
        "sha": "'$GIT_SHA'",
        "environment": "production",
        "deployer": "argocd",
        "timestamp": "'$(date -Iseconds)'"
      }'
```

---

## Top 3 Tools for Out-of-the-Box DORA Metrics

These tools work **regardless of your CI/CD and code repo tooling** — they sit outside the pipeline and observe it.

### 1. Sleuth (sleuth.io) — Best Overall

**Why**: Purpose-built for DORA. Fastest time-to-value.

| Aspect | Detail |
|--------|--------|
| **Setup time** | < 1 day |
| **Integrations** | GitHub, GitLab, Bitbucket, GHA, Jenkins, CircleCI, ArgoCD, Harness, Spinnaker, PagerDuty, Jira, Opsgenie, LaunchDarkly |
| **How it works** | You register a "deploy" (webhook, CI event, CD sync). Sleuth auto-correlates commits via SHA and links incidents from your alerting tool. |
| **DORA coverage** | All 4 metrics, real-time dashboard, team/service breakdown |
| **Pricing** | Free tier (1 project), Growth $29/dev/mo, Enterprise custom |
| **Differentiator** | Deploy verification — automatically checks error rates, latency after deploy to flag failures without waiting for an incident |

```
How Sleuth connects your stack:

  GitHub ──────┐
  GHA ─────────┼──> Sleuth ──> DORA Dashboard
  ArgoCD ──────┤       │
  PagerDuty ───┘       └──> Slack/Email alerts on regression
```

**Best for**: Teams that want DORA metrics working in hours, not weeks.

---

### 2. Faros AI (faros.ai) — Best for Enterprise / Multi-Tool

**Why**: 100+ connectors, unified data model, works with any combination of tools.

| Aspect | Detail |
|--------|--------|
| **Setup time** | 1-3 days |
| **Integrations** | 100+ connectors: every major SCM, CI, CD, issue tracker, incident mgmt, deployment tool |
| **How it works** | Connectors pull data from each tool into a normalized graph database. DORA metrics computed automatically from the unified model. Custom metrics via GraphQL. |
| **DORA coverage** | All 4 + engineering metrics (rework rate, review time, PR cycle time) |
| **Pricing** | Community Edition (open-source, self-hosted), Enterprise (managed) |
| **Differentiator** | Works with literally any toolchain — if it has an API, Faros can ingest it |

```
How Faros normalizes your stack:

  GitHub ──────┐                    ┌──> DORA Metrics
  GHA ─────────┤                    ├──> Engineering Metrics
  Artifact Reg ┼──> Faros Graph ────┤
  ArgoCD ──────┤    (normalized)    ├──> Custom Dashboards
  Harness ─────┤                    ├──> Jira/Linear sync
  PagerDuty ───┘                    └──> API / GraphQL
```

**Best for**: Large organizations with 5+ tools in the delivery pipeline, or those needing metrics beyond DORA.

---

### 3. Four Keys (dora.dev/fourkeys) — Best Open-Source / GCP-Native

**Why**: Built by the DORA team at Google. Event-driven, fully customizable, free.

| Aspect | Detail |
|--------|--------|
| **Setup time** | 2-5 days |
| **Integrations** | Any tool that can send a webhook (tool-agnostic by design) |
| **How it works** | Webhooks → Cloud Function (event parser) → BigQuery (data store) → Looker (dashboard). You write parsers for each event type. |
| **DORA coverage** | All 4 metrics with reference Looker dashboards |
| **Pricing** | Free (open-source), GCP infra costs only (~$20-50/mo) |
| **Differentiator** | Complete control, no vendor lock-in, endorsed by the DORA team |

```
Architecture:

  GitHub webhook ───────┐
  GHA webhook ──────────┤
  ArgoCD notification ──┼──> Cloud Function ──> BigQuery ──> Looker Dashboard
  PagerDuty webhook ────┤    (event parser)     (storage)    (visualization)
  Harness webhook ──────┘
```

**Best for**: GCP-native teams, or those wanting full control and no vendor dependency.

---

## Tool Comparison Matrix

| Criteria | Sleuth | Faros AI | Four Keys |
|----------|--------|----------|-----------|
| **Time to value** | Hours | Days | Days-Week |
| **Tool-agnostic** | Yes (40+ integrations) | Yes (100+ connectors) | Yes (webhook-based) |
| **Self-hosted option** | No | Yes (Community Edition) | Yes (GCP only) |
| **Beyond DORA** | Deploy verification | Full eng metrics | No (DORA only) |
| **Maintenance** | Zero (SaaS) | Low-Medium | Medium (own infra) |
| **Cost** | $29/dev/mo | Free (CE) / Enterprise | GCP infra only |
| **Best for** | Fast start, mid-size teams | Enterprise, complex toolchains | GCP-native, full control |
| **GitHub + ArgoCD** | Native support | Native connectors | Custom webhook parser |

---

## Implementation Roadmap

### Week 1: Instrument Your Pipeline

```
1. Tag every container image with the commit SHA
   docker build -t myapp:${GITHUB_SHA} .

2. Ensure your CD tool (ArgoCD/Harness) records:
   - What SHA was deployed
   - When the deployment happened
   - Whether it succeeded

3. Link your incident management tool:
   - PagerDuty: Tag incidents with service + deployment SHA
   - Jira: Add "deployment_sha" custom field to incident issues
```

### Week 2: Choose and Set Up Your Tool

```
Option A (Fastest): Sleuth
  1. Sign up → Connect GitHub → Connect ArgoCD → Connect PagerDuty
  2. Define your deploy sources (ArgoCD sync = deployment)
  3. Dashboard shows DORA metrics immediately

Option B (Enterprise): Faros AI
  1. Deploy Faros Community Edition (Docker Compose)
  2. Configure connectors: GitHub, GHA, ArgoCD, PagerDuty
  3. Run initial sync, review auto-computed DORA metrics
  4. Customize dashboards via GraphQL

Option C (Open-source): Four Keys
  1. Deploy to GCP (Terraform module provided)
  2. Configure webhooks from GitHub, ArgoCD, PagerDuty
  3. Write event parsers for your specific event format
  4. Import reference Looker dashboards
```

### Week 3: Baseline and Iterate

```
1. Run for 2 weeks to establish baseline
2. Identify which DORA band you're in (Elite/High/Medium/Low)
3. Pick ONE metric to improve first (usually Lead Time — it has the biggest downstream impact)
4. Set quarterly targets (move one band per quarter is realistic)
```

---

## Quick Reference: API Endpoints

### GitHub Actions

```bash
# List workflow runs (deploys)
gh api repos/{owner}/{repo}/actions/runs --jq '.workflow_runs[] | {sha: .head_sha, status: .conclusion, time: .updated_at}'

# Workflow run timing
gh api repos/{owner}/{repo}/actions/runs/{run_id}/timing

# Per-job timing (queue time = started_at - created_at)
gh api repos/{owner}/{repo}/actions/runs/{run_id}/jobs --jq '.jobs[] | {name: .name, queued: .created_at, started: .started_at, completed: .completed_at}'
```

### ArgoCD

```bash
# List application syncs
argocd app history myapp-prod -o json | jq '.[] | {revision: .revision, deployedAt: .deployedAt, status: .status}'

# Sync events via API
curl -s -H "Authorization: Bearer $ARGOCD_TOKEN" \
  https://argocd.example.com/api/v1/applications/myapp-prod \
  | jq '.status.history[] | {revision: .revision, deployedAt: .deployedAt}'
```

### PagerDuty

```bash
# Incidents for MTTR
curl -s -H "Authorization: Token token=$PD_TOKEN" \
  "https://api.pagerduty.com/incidents?since=2026-08-01&until=2026-09-01&statuses[]=resolved" \
  | jq '.incidents[] | {
      service: .service.summary,
      created: .created_at,
      resolved: .last_status_change_at,
      mttr_minutes: (((.last_status_change_at | fromdateiso8601) - (.created_at | fromdateiso8601)) / 60)
    }'
```

---

## Related Resources

- [DORA Quick Check](https://dora.dev/quickcheck/) — Self-assessment quiz
- [Accelerate Book](https://itrevolution.com/product/accelerate/) — The foundational research
- [DORA Core Model](https://dora.dev/research/) — Latest research findings
- [Sleuth Docs](https://help.sleuth.io/) — Setup guides
- [Faros Community](https://github.com/faros-ai/faros-community-edition) — Self-hosted setup
- [Four Keys](https://github.com/dora-team/fourkeys) — Google's open-source implementation
