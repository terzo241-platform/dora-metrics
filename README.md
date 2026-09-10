# DORA Metrics: Practical Implementation Guide

> How to measure software delivery performance across fragmented toolchains — updated for 2026 with honest assessments of what works, what doesn't, and what no tool solves for you.

---

## Why DORA Metrics Matter

DORA (DevOps Research and Assessment) metrics are the industry standard for measuring software delivery performance. The [Accelerate State of DevOps Report](https://dora.dev) — backed by 10+ years of research across 39,000+ professionals — consistently shows that **elite performers deploy 200x more frequently with 2,555x faster lead times** than low performers.

These aren't vanity metrics. Teams that improve on DORA metrics see direct impact on:
- **Business outcomes**: Revenue, market share, customer satisfaction
- **Operational health**: Less burnout, lower unplanned work (22% less for elite teams)
- **Organizational performance**: 2x likelihood of exceeding goals

---

## The Five Metrics (Updated 2026)

As of January 2026, DORA officially tracks **5 metrics** (not 4). The addition of Deployment Rework Rate reflects the reality that rollbacks and hotfixes are a distinct signal from change failure rate.

| Metric | What It Measures | Elite | High | Medium | Low |
|--------|-----------------|-------|------|--------|-----|
| **Deployment Frequency** | How often you deploy to production | Many per day | Daily to weekly | Weekly to monthly | Monthly+ |
| **Change Lead Time** | Commit to running in production | < 1 day | 1 day - 1 week | 1 week - 1 month | > 1 month |
| **Change Fail Rate** | % of deployments needing rollback/hotfix | 5% | 10% | 15% | 64% |
| **Failed Deployment Recovery Time** | Time to recover from a failed deployment | < 1 hour | < 1 day | 1 day - 1 week | 1 month+ |
| **Deployment Rework Rate** (NEW) | Ratio of unplanned deploys caused by prod incidents | Low | Moderate | High | Very High |

> **Note**: "Failed Deployment Recovery Time" replaces the older "MTTR" terminology. "Deployment Rework Rate" is new — it captures how much of your deployment activity is reactive firefighting vs planned delivery.

---

## The Real-World Problem: Not Every Commit Goes Straight to Prod

DORA's theoretical model assumes continuous delivery — every commit flows to production independently. **This is not how most enterprises work.** The typical enterprise flow looks like this:

```
Developer commits (Day 1)
    │
    ├── Multiple developers commit to feature branches (Days 1-10)
    │
    ▼
Merge to develop/integration branch (Day 10)
    │
    ├── Automated tests run (CI)
    │
    ▼
Code freeze announced (Day 12)
    │
    ├── Release branch cut
    │
    ▼
QA/UAT cycle begins (Days 12-18)
    │
    ├── Manual testing (regression, UAT sign-off)
    ├── Bug fixes cherry-picked to release branch
    │
    ▼
Production deployment (Day 20)
    │
    └── 30+ commits deployed as a single batch
```

**The hard truth**: DORA's official guidance does NOT address how to compute lead time for batched releases. Their position is philosophical — reduce batch size, move toward continuous delivery. But you need metrics for where you ARE today, not where you want to be.

---

## How Lead Time Actually Works for Batched Releases

### The Three Approaches (With Honest Trade-offs)

#### Approach 1: Per-Commit Lead Time (DORA Purist)

```
Lead Time = T_prod_deploy - T_commit_authored

For a release with 30 commits:
  - Commit A authored Day 1  → deployed Day 20 → Lead Time = 19 days
  - Commit B authored Day 5  → deployed Day 20 → Lead Time = 15 days
  - Commit Z authored Day 10 → deployed Day 20 → Lead Time = 10 days

Report: median = 15 days, p95 = 19 days
```

**Pros**: True DORA definition. Exposes how long developers actually wait.
**Cons**: Penalizes long-lived feature branches. A commit authored Day 1 but not merged until Day 9 shows as 19 days even though it was "active" for only 1 day.

#### Approach 2: Per-MR/PR Lead Time (GitLab's Approach — Industry Standard)

```
Lead Time = T_prod_deploy - T_merge_to_main

For a release with 30 commits (merged as 12 PRs):
  - PR #101 merged Day 8  → deployed Day 20 → Lead Time = 12 days
  - PR #105 merged Day 10 → deployed Day 20 → Lead Time = 10 days
  - PR #112 merged Day 12 → deployed Day 20 → Lead Time = 8 days

Report: median = 10 days
```

**Pros**: Pragmatic. Captures the "system wait time" — how long merged, reviewed code sits before reaching production. This is the delay the platform team can actually influence.
**Cons**: Hides development time within the feature branch.

#### Approach 3: Release-Level Lead Time (Common Enterprise Shortcut)

```
Lead Time = T_prod_deploy - T_code_freeze

Code freeze Day 12 → deployed Day 20 → Lead Time = 8 days
```

**Pros**: Simple. Measures QA/release process overhead directly.
**Cons**: Hides everything before code freeze. NOT a valid DORA metric — but useful as an internal operational metric.

### Recommendation for Enterprise Platform Teams

**Use Approach 2 (per-MR) as your primary DORA metric**, supplemented with Approach 3 as an operational metric.

Why: Per-MR lead time is the one metric that a centralized platform team can actually influence (faster CI, faster deploys, better environments). Development time within feature branches is a team-level concern, not a platform concern.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LEAD TIME DECOMPOSITION                          │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌───────┐  │
│  │  Coding  │─>│  Review  │─>│  Merge   │─>│   QA   │─>│Deploy │  │
│  │  Time    │  │  Time    │  │  to main │  │  Cycle │  │to Prod│  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  └───────┘  │
│  ◄── Team owns ──────────────►◄── Platform team owns ───────────► │
│                                                                     │
│  Sleuth tracks these phases:                                        │
│  Coding → Review Lag → Review Time → Deploying                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How to Track Manual Testing in DORA Metrics

**Honest assessment: No DORA tool tracks manual testing natively.** Not Sleuth, not Faros, not the archived Four Keys, not GitLab. This is a universal gap.

Manual testing time is *implicitly* captured in lead time (it's part of the delay between merge and deploy), but it's not broken out as a sub-metric by any standard tool. You have to instrument it yourself.

### Pattern: Jira Workflow Timestamps

The most practical approach is to use your issue tracker's workflow transitions as stage markers:

```
Jira Workflow for Release:
  ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
  │  To Do   │───>│ In Dev    │───>│ In QA    │───>│ QA Passed │───>│ Released │
  │          │    │           │    │          │    │           │    │          │
  │ T_create │    │ T_dev     │    │ T_qa     │    │ T_passed  │    │ T_deploy │
  └──────────┘    └───────────┘    └──────────┘    └───────────┘    └──────────┘

QA Cycle Time = T_passed - T_qa
Total Lead Time = T_deploy - T_dev (or T_merge)
QA as % of Lead Time = QA Cycle Time / Total Lead Time * 100
```

### Pattern: Test Management Tool Integration

If you use Zephyr, TestRail, or qTest for manual test execution:

```
Capture from test management API:
  - Test cycle start time (when QA begins execution)
  - Test cycle end time (when all test cases pass/fail)
  - Number of defects found → fed back as bug fix cycle time
  - Re-test cycles (how many rounds before sign-off)

TestRail API example:
  GET /index.php?/api/v2/get_runs/{project_id}
  → .runs[] | {started_on, completed_on, failed_count, passed_count}
```

### Pattern: GitHub Deployment Environments as Stage Gates

Use GitHub's native environment protection rules to create observable QA stages:

```yaml
# .github/workflows/release.yml
jobs:
  deploy-staging:
    environment: staging           # ← timestamp recorded by GitHub
    steps:
      - run: deploy-to-staging.sh

  manual-qa:                       
    needs: deploy-staging
    environment: manual-qa         # ← requires manual approval in GitHub
    steps:                         # ← approval timestamp = QA start
      - run: echo "QA approved"    # ← completion timestamp = QA end

  deploy-production:
    needs: manual-qa
    environment: production        # ← production deploy timestamp
    steps:
      - run: deploy-to-prod.sh
```

```bash
# Extract QA cycle time from GitHub Environments API
gh api repos/{owner}/{repo}/actions/runs/{run_id}/pending_deployments
# Shows: environment, wait_timer, reviewers, current_status

# After approval:
gh api repos/{owner}/{repo}/actions/runs/{run_id}/approvals
# Shows: environment, approved_at (= QA sign-off timestamp)
```

This gives you **observable timestamps for manual QA** without any custom tooling — GitHub records when the approval was requested (QA started) and when it was granted (QA passed).

---

## Handling Different Branching Strategies Across Teams

### The Reality

In any large organization, teams use different strategies:

| Strategy | Who Uses It | How Releases Work |
|----------|-------------|-------------------|
| **Trunk-based** | Mature DevOps teams | Every merge to main → auto-deploy |
| **GitFlow** | Teams with release cycles | develop → release branch → main → deploy |
| **Release branches** | Teams with QA gates | main → release/v1.2 → QA → deploy |
| **Environment branches** | Legacy teams | dev branch → staging branch → prod branch |

### The Platform Team's Normalization Strategy

**Don't force teams into one branching strategy. Normalize at the measurement layer.**

The key insight: regardless of branching strategy, every team has two events you can observe:

1. **Code is "done"** = PR/MR merged to the team's integration branch (whatever they call it)
2. **Code reaches production** = deployment event from CD tool

```
                    ┌─────────────────────────────────────┐
                    │    NORMALIZATION LAYER               │
                    │                                      │
  Trunk-based:     │  merge to main ──────────> prod      │  Lead Time = hours
  GitFlow:         │  merge to develop ────────> prod     │  Lead Time = days
  Release branch:  │  merge to release/X ─────> prod     │  Lead Time = days
  Env branches:    │  merge to staging ────────> prod     │  Lead Time = weeks
                    │                                      │
                    │  ALL measured as:                     │
                    │  T_prod_deploy - T_merge_event       │
                    └─────────────────────────────────────┘
```

### Implementation: Branch-Agnostic Event Collection

Configure your metrics platform to watch **deployment events only** (not branch merges) as the primary signal:

```yaml
# Platform-level configuration per team
teams:
  - name: payments
    repos: [payments-api, payments-ui]
    deploy_detection: argocd_sync     # ArgoCD app name = payments-prod
    merge_branch: main                # Trunk-based
    
  - name: inventory  
    repos: [inventory-service]
    deploy_detection: harness_pipeline # Harness pipeline = inventory-prod-deploy
    merge_branch: develop             # GitFlow — track merge to develop
    
  - name: reporting
    repos: [reports-engine]
    deploy_detection: gha_workflow     # GHA workflow = deploy-production
    merge_branch: release/*           # Release branches — track merge to release/*
```

The **deployment event** is the universal anchor — it comes from the CD tool and is the same regardless of branching strategy. Work backward from the deploy to find the associated commits/PRs.

---

## Four Keys: Honest Assessment (Why You Shouldn't Use It As-Is)

### Status: Archived January 2024

The `dora-team/fourkeys` GitHub repository is **archived and read-only**. The README states:

> *"This repository is not currently maintained. We encourage you to explore it, fork it, or otherwise use it as inspiration."*

**No GCP-native replacement exists.** Google Cloud Deploy does NOT have built-in DORA metrics dashboards. The DORA team at Google now recommends the [DORA Quick Check](https://dora.dev/quickcheck/) self-assessment and *"source-available or commercial products with pre-built integrations"* rather than custom-built pipelines.

### Why Four Keys Fails for Enterprise Batched Releases

Four Keys was built on assumptions that don't hold in enterprise environments:

| Assumption | Reality |
|-----------|---------|
| Every merge to main triggers a deployment | Batched releases, code freezes, manual approvals |
| Commit SHA maps 1:1 to a deployment | 30+ commits per release, squash merges break SHA links |
| No intermediate stages between merge and deploy | UAT, manual QA, change advisory boards, release trains |
| All teams use the same branching strategy | GitFlow, trunk-based, release branches coexist |
| Deployments are atomic per-repo | Multi-repo deployments, mono-repo with partial deploys |

The Four Keys BigQuery SQL directly correlated merge events to deploy events by SHA. If your flow has staging/UAT/manual QA between merge and prod, there is no hook for those intermediate stages.

### What's Still Valuable from Four Keys

The **architectural pattern** is sound even though the code is abandoned:

```
Events (webhooks) → Ingestion (Cloud Run/Function) → Storage (BigQuery) → Dashboard (Looker/Grafana)
```

If you want to build on GCP, fork the pattern (not the code) and add:
- Stage-aware event model (not just merge → deploy)
- Release entity linking multiple commits/PRs to a single deployment
- QA stage timestamps from Jira/test management tools
- Team-level configuration for branching strategy normalization

---

## Recommended Approach for a Centralized Platform Team

### The Platform DORA Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TEAM TOOLCHAINS (Diverse)                        │
│                                                                         │
│  Team A: GitHub → GHA → ArgoCD → Prod    (trunk-based)                 │
│  Team B: GitHub → GHA → Harness → Prod   (GitFlow)                    │
│  Team C: GitLab → Jenkins → ArgoCD → Prod (release branches)          │
│  Team D: GitHub → GHA → GHA Deploy → Prod (trunk-based)               │
└────────┬───────────────────┬──────────────────┬────────────────────┬────┘
         │                   │                  │                    │
    Webhooks/APIs       Webhooks/APIs      Webhooks/APIs       Webhooks/APIs
         │                   │                  │                    │
         ▼                   ▼                  ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    NORMALIZATION LAYER (Platform-Owned)                  │
│                                                                         │
│  Event Ingestion    ───>  Canonical Data Model  ───>  Metric Compute   │
│  (Cloud Run)              (BigQuery)                  (Scheduled Query) │
│                                                                         │
│  Canonical Events:                                                      │
│  ┌────────────────────────────────────────────────────────────┐         │
│  │ { team, repo, event_type, sha, branch, timestamp,         │         │
│  │   environment, release_id, metadata }                      │         │
│  │                                                            │         │
│  │ event_type: commit | pr_merged | build_complete |          │         │
│  │             qa_started | qa_passed | deployed |            │         │
│  │             incident_opened | incident_resolved            │         │
│  └────────────────────────────────────────────────────────────┘         │
│                                                                         │
│  Release Entity (the missing piece in Four Keys):                       │
│  ┌────────────────────────────────────────────────────────────┐         │
│  │ { release_id, team, repo, environment,                     │         │
│  │   commits: [sha1, sha2, ...],                              │         │
│  │   prs: [pr1, pr2, ...],                                    │         │
│  │   code_freeze_at, qa_start_at, qa_end_at, deployed_at }   │         │
│  └────────────────────────────────────────────────────────────┘         │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    DASHBOARD LAYER                                       │
│                                                                         │
│  Org-level:  All 5 DORA metrics, trend over time, band classification  │
│  Team-level: Per-team DORA + lead time decomposition                   │
│  Drill-down: Per-release breakdown (coding → review → QA → deploy)    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Tool Recommendation: Revised (Post-Research)

Given that Four Keys is archived and your scenario involves batched releases + manual QA + diverse branching:

| Criteria | Sleuth | Faros AI CE | Build-Your-Own (GCP) |
|----------|--------|-------------|----------------------|
| **Handles batched releases** | Yes (per-deploy averaging) | Yes (normalized model) | Yes (if you build it) |
| **Manual QA tracking** | No (manual gap) | Partial (Jira connector) | Yes (custom events) |
| **Diverse branching** | Partial (deploy-anchored) | Yes (100+ connectors) | Yes (configurable) |
| **Time to value** | Hours | 1-3 days | 2-4 weeks |
| **Maintenance** | Zero | Low | Medium-High |
| **Cost (100 devs)** | ~$2,900/mo | Free (self-hosted) | GCP infra ~$50-100/mo |
| **Customization** | Limited | GraphQL queries | Unlimited |
| **Air-gapped / on-prem** | No | Yes | Yes |

### Honest Recommendation

**For most enterprise platform teams: Start with Sleuth or Faros AI CE to get baseline metrics in days, not months.** Use the data to prove value to leadership. Then decide whether to build custom tooling for the gaps (manual QA tracking, release-level decomposition).

**If you must build on GCP** (compliance, data residency, existing GCP investment):

Fork the Four Keys *pattern* (not the archived code) with these additions:
1. **Release entity** linking commits/PRs to deployments
2. **Stage events** for QA (from Jira workflow transitions)
3. **Team configuration** for branching strategy normalization
4. **5 metrics** (not 4 — add Deployment Rework Rate)

---

## Build-Your-Own on GCP: Practical Blueprint

If you choose to build this as a platform service on GCP, here's the architecture that addresses all the gaps:

### Component 1: Event Ingestion (Cloud Run)

```python
# Receives webhooks from GitHub, ArgoCD, Jira, PagerDuty
# Normalizes into canonical events and writes to BigQuery

CANONICAL_EVENT_SCHEMA = {
    "event_id": "string",
    "timestamp": "timestamp",
    "team": "string",
    "repo": "string",
    "event_type": "string",     # commit, pr_merged, build, qa_started, qa_passed, deployed, incident_opened, incident_resolved
    "sha": "string",
    "branch": "string",
    "environment": "string",    # dev, staging, uat, production
    "release_id": "string",     # groups commits into a release
    "metadata": "json",         # tool-specific payload
}
```

### Component 2: Release Linker (Scheduled Cloud Function)

This is the piece Four Keys never had — it groups commits into releases:

```sql
-- BigQuery: Link commits to releases
-- A "release" is defined as all commits between two production deployments

WITH deployments AS (
  SELECT 
    team, repo, sha, timestamp as deployed_at,
    LAG(timestamp) OVER (PARTITION BY team, repo ORDER BY timestamp) as prev_deploy_at
  FROM events
  WHERE event_type = 'deployed' AND environment = 'production'
),
release_commits AS (
  SELECT 
    d.team, d.repo, d.deployed_at,
    c.sha as commit_sha,
    c.timestamp as committed_at,
    TIMESTAMP_DIFF(d.deployed_at, c.timestamp, HOUR) as lead_time_hours
  FROM deployments d
  JOIN events c ON c.team = d.team AND c.repo = d.repo
    AND c.event_type = 'pr_merged'
    AND c.timestamp > COALESCE(d.prev_deploy_at, TIMESTAMP('1970-01-01'))
    AND c.timestamp <= d.deployed_at
)
SELECT 
  team, repo, deployed_at,
  COUNT(commit_sha) as commits_in_release,
  AVG(lead_time_hours) as avg_lead_time_hours,
  MAX(lead_time_hours) as max_lead_time_hours,    -- oldest commit in batch
  MIN(lead_time_hours) as min_lead_time_hours     -- newest commit in batch
FROM release_commits
GROUP BY team, repo, deployed_at
```

### Component 3: QA Cycle Tracking (Jira Webhook Parser)

```python
# Parse Jira webhook for workflow transitions
def parse_jira_event(payload):
    changelog = payload.get("changelog", {}).get("items", [])
    for item in changelog:
        if item["field"] == "status":
            if item["toString"] == "In QA":
                return {
                    "event_type": "qa_started",
                    "release_id": extract_release_from_jira(payload),
                    "timestamp": payload["timestamp"],
                    "team": extract_team(payload),
                }
            elif item["toString"] == "QA Passed":
                return {
                    "event_type": "qa_passed",
                    "release_id": extract_release_from_jira(payload),
                    "timestamp": payload["timestamp"],
                    "team": extract_team(payload),
                }
```

### Component 4: DORA Computation (Scheduled BigQuery Query)

```sql
-- All 5 DORA metrics computed from canonical events

-- 1. Deployment Frequency (per team, per week)
SELECT team, DATE_TRUNC(timestamp, WEEK) as week,
  COUNT(*) as deploy_count
FROM events
WHERE event_type = 'deployed' AND environment = 'production'
GROUP BY team, week;

-- 2. Change Lead Time (per team, per week — using PR merge as start)
SELECT team, DATE_TRUNC(deployed_at, WEEK) as week,
  APPROX_QUANTILES(lead_time_hours, 100)[OFFSET(50)] as p50_lead_time_hours,
  APPROX_QUANTILES(lead_time_hours, 100)[OFFSET(95)] as p95_lead_time_hours
FROM release_commits_view
GROUP BY team, week;

-- 3. Change Fail Rate
WITH deploys AS (
  SELECT team, DATE_TRUNC(timestamp, WEEK) as week, COUNT(*) as total
  FROM events WHERE event_type = 'deployed' AND environment = 'production'
  GROUP BY team, week
),
failures AS (
  SELECT team, DATE_TRUNC(timestamp, WEEK) as week, COUNT(*) as failed
  FROM events WHERE event_type = 'incident_opened'
  GROUP BY team, week
)
SELECT d.team, d.week, 
  SAFE_DIVIDE(f.failed, d.total) * 100 as change_fail_rate_pct
FROM deploys d LEFT JOIN failures f ON d.team = f.team AND d.week = f.week;

-- 4. Failed Deployment Recovery Time
SELECT team, DATE_TRUNC(i.timestamp, WEEK) as week,
  AVG(TIMESTAMP_DIFF(r.timestamp, i.timestamp, MINUTE)) as avg_recovery_minutes
FROM events i
JOIN events r ON i.team = r.team 
  AND r.event_type = 'incident_resolved'
  AND r.metadata->>'incident_id' = i.metadata->>'incident_id'
WHERE i.event_type = 'incident_opened'
GROUP BY team, week;

-- 5. Deployment Rework Rate (NEW in DORA 2026)
WITH all_deploys AS (
  SELECT team, DATE_TRUNC(timestamp, WEEK) as week, COUNT(*) as total
  FROM events WHERE event_type = 'deployed' AND environment = 'production'
  GROUP BY team, week
),
unplanned_deploys AS (
  SELECT team, DATE_TRUNC(timestamp, WEEK) as week, COUNT(*) as unplanned
  FROM events 
  WHERE event_type = 'deployed' AND environment = 'production'
    AND JSON_VALUE(metadata, '$.is_hotfix') = 'true'
  GROUP BY team, week
)
SELECT a.team, a.week,
  SAFE_DIVIDE(u.unplanned, a.total) * 100 as rework_rate_pct
FROM all_deploys a LEFT JOIN unplanned_deploys u ON a.team = u.team AND a.week = u.week;
```

### Component 5: QA Sub-Metrics (Bonus — Not Standard DORA)

```sql
-- QA cycle time as a sub-metric of Lead Time
SELECT team, DATE_TRUNC(qa_started, WEEK) as week,
  AVG(TIMESTAMP_DIFF(qa_passed, qa_started, HOUR)) as avg_qa_hours,
  COUNT(*) as releases_tested
FROM (
  SELECT 
    s.team,
    s.timestamp as qa_started,
    p.timestamp as qa_passed,
    s.release_id
  FROM events s
  JOIN events p ON s.release_id = p.release_id 
    AND s.team = p.team
    AND s.event_type = 'qa_started' 
    AND p.event_type = 'qa_passed'
)
GROUP BY team, week;
```

---

## How Big Organizations Handle This Problem

### The Uncomfortable Truth

**No published case studies from Google, Netflix, or Spotify (2025-2026) detail how they run DORA as a centralized platform service.** The Spotify Backstage DORA plugin exists in community discussions but has no official documentation. Google's own DORA team archived their only tool.

What we know from the DORA 2024 report:

> *"Utilizing an internal developer platform improves individual productivity, team performance, and organizational performance"* — but cautions that *"poorly implemented platforms can reduce change stability and throughput."*

### What Actually Works (From Enterprise Practitioners)

Based on verified implementations and tool documentation:

**1. Standardize on the deployment event, not the branching model**

Every team, regardless of branching strategy, has a moment when code reaches production. Anchor your metrics to that event. It comes from your CD tool (ArgoCD sync, Harness pipeline completion, GHA deployment workflow) and is consistent across teams.

**2. Provide a "team configuration" layer**

Let each team declare:
- Their repos
- Their CD tool and deploy detection method
- Their integration branch (main, develop, release/*)
- Their QA process (automated-only, manual, or hybrid)

The platform normalizes the rest.

**3. Separate DORA metrics from operational sub-metrics**

| Layer | Metrics | Audience | Standard |
|-------|---------|----------|----------|
| **DORA (org-level)** | 5 DORA metrics per team | Leadership, platform team | Industry standard |
| **Operational (team-level)** | QA cycle time, review time, build time, queue time | Team leads, engineers | Internal standard |
| **Diagnostic (drill-down)** | Per-release, per-PR, per-stage breakdown | Engineers debugging | Ad-hoc |

**4. Accept that lead time will look bad — that's the point**

For teams doing 2-week sprints with manual QA, lead time will be 2-4 weeks. That's not a failure of measurement — it's an accurate picture of reality. The metric is designed to make the case for investment in automation and continuous delivery.

**5. Don't fake continuous delivery metrics onto a batched release process**

If a team deploys biweekly, their deployment frequency is "biweekly." Don't count deployments to staging or UAT as "deployments" to inflate the number. Honest measurement is the prerequisite for honest improvement.

---

## Implementation Roadmap for Platform Teams

### Phase 1: Deploy Event Capture (Week 1-2)

Get the deployment signal right first. Everything else builds on this.

```
For each CD tool in use:
  ArgoCD → Enable notifications → webhook to your ingestion endpoint
  Harness → Pipeline event webhook → your endpoint  
  GHA     → Repository webhook (workflow_run events) → your endpoint

Validate: Can you answer "how many production deployments happened last week per team?"
```

### Phase 2: Commit/PR Linkage (Week 2-3)

Connect deployments to the code changes they contain.

```
For each deployment event:
  1. Get the deployed SHA/image tag
  2. Query GitHub API for all commits between this deploy and the previous one
  3. Query GitHub API for the PRs that introduced those commits
  4. Store the mapping: deploy → [PR1, PR2, ...] → [commit1, commit2, ...]

Validate: Can you answer "what PRs were in last Tuesday's deployment?"
```

### Phase 3: Incident Linkage (Week 3-4)

Connect incidents to deployments.

```
For PagerDuty/Jira/Opsgenie:
  1. Webhook on incident create/resolve
  2. Link to deployment via: time proximity OR explicit tagging
  3. Classify: was this caused by a deployment or unrelated?

Validate: Can you compute Change Fail Rate for last month?
```

### Phase 4: QA Stage Instrumentation (Week 4-5)

Add manual QA visibility (the gap no tool fills for you).

```
Option A: Jira workflow webhooks (qa_started / qa_passed transitions)
Option B: GitHub Environment approvals (staging → manual-qa → production)
Option C: Test management API (TestRail/Zephyr test cycle start/end)

Validate: Can you answer "how long did QA take for the last 5 releases?"
```

### Phase 5: Dashboard and Reporting (Week 5-6)

```
Build Grafana/Looker dashboards at three levels:
  1. Org overview: 5 DORA metrics, all teams, trend over time
  2. Team detail: per-team DORA + lead time decomposition
  3. Release drill-down: specific release → stages → bottleneck identification

Validate: Can leadership see which teams are improving and which are stuck?
```

### Phase 6: Customization Layer (Ongoing)

```
Allow teams to self-configure:
  - Add/remove repos
  - Set their branching strategy  
  - Configure deploy detection
  - Set QA process flags

Platform team maintains:
  - Event ingestion infrastructure
  - Canonical data model
  - Dashboard templates
  - Metric computation logic
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

# Deployment environment approvals (for QA stage tracking)
gh api repos/{owner}/{repo}/actions/runs/{run_id}/approvals
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
# Incidents for Recovery Time
curl -s -H "Authorization: Token token=$PD_TOKEN" \
  "https://api.pagerduty.com/incidents?since=2026-08-01&until=2026-09-01&statuses[]=resolved" \
  | jq '.incidents[] | {
      service: .service.summary,
      created: .created_at,
      resolved: .last_status_change_at,
      recovery_minutes: (((.last_status_change_at | fromdateiso8601) - (.created_at | fromdateiso8601)) / 60)
    }'
```

---

## Related Resources

- [DORA Quick Check](https://dora.dev/quickcheck/) — Self-assessment quiz (recommended starting point)
- [DORA Core Model](https://dora.dev/research/) — Latest research findings (updated Jan 2026)
- [Accelerate Book](https://itrevolution.com/product/accelerate/) — The foundational research
- [Sleuth Docs](https://help.sleuth.io/) — Setup guides
- [Faros Community Edition](https://github.com/faros-ai/faros-community-edition) — Self-hosted setup
- [Four Keys (archived)](https://github.com/dora-team/fourkeys) — Archived Jan 2024, useful as architectural reference only
