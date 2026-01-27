# Canary Deployment Guide - Voting App QA Environment

## Table of Contents
1. [What is Canary Deployment?](#what-is-canary-deployment)
2. [Why Canary Over Other Strategies?](#why-canary-over-other-strategies)
3. [Voting App Canary Architecture](#voting-app-canary-architecture)
4. [Step-by-Step Flow Simulation](#step-by-step-flow-simulation)
5. [Analysis & Auto-Rollback](#analysis--auto-rollback)
6. [Manual Operations](#manual-operations)
7. [Monitoring & Observability](#monitoring--observability)
8. [Troubleshooting Guide](#troubleshooting-guide)

---

## What is Canary Deployment?

### The Concept

**Canary deployment** is named after the "canary in a coal mine" practice where miners used canaries to detect toxic gases. If the canary showed signs of distress, miners knew to evacuate before they were affected.

In software deployment, the **canary** is a small subset of users who receive the new version first. If something goes wrong, only this small group is affected, and the system can quickly rollback before impacting all users.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CANARY DEPLOYMENT CONCEPT                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Traditional Deployment (Big Bang):                                   │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │  100% traffic ──────────────────────────► New Version       │      │
│   │                                           (All or Nothing)  │      │
│   └─────────────────────────────────────────────────────────────┘      │
│   Risk: If new version has bugs, ALL users are affected immediately   │
│                                                                         │
│   Canary Deployment (Progressive):                                     │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │  10% traffic  ───► New Version (Canary)    ◄── Monitor      │      │
│   │  90% traffic  ───► Old Version (Stable)                     │      │
│   │                         │                                   │      │
│   │                         ▼                                   │      │
│   │              If healthy, increase to 30%                    │      │
│   │              If unhealthy, ROLLBACK to 0%                   │      │
│   └─────────────────────────────────────────────────────────────┘      │
│   Risk: Only 10% users see issues, 90% remain unaffected              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Reduced Blast Radius** | Only a small % of users affected if issues occur |
| **Real Production Testing** | Test with real traffic, not just synthetic tests |
| **Automated Rollback** | System detects issues and reverts automatically |
| **Gradual Confidence** | Build confidence as more traffic shifts successfully |
| **Zero Downtime** | Users never experience service interruption |

---

## Why Canary Over Other Strategies?

### Deployment Strategy Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   DEPLOYMENT STRATEGIES COMPARISON                      │
├──────────────┬─────────────┬─────────────┬─────────────┬───────────────┤
│   Strategy   │   Risk      │  Rollback   │ Resource    │  Complexity   │
│              │   Level     │   Speed     │ Usage       │               │
├──────────────┼─────────────┼─────────────┼─────────────┼───────────────┤
│ Recreate     │ ████████    │ ████████    │ █           │ █             │
│ (Replace)    │ Very High   │ Very Slow   │ Low         │ Simple        │
├──────────────┼─────────────┼─────────────┼─────────────┼───────────────┤
│ Rolling      │ █████       │ █████       │ ██          │ ██            │
│ Update       │ Medium      │ Medium      │ Medium      │ Moderate      │
├──────────────┼─────────────┼─────────────┼─────────────┼───────────────┤
│ Blue-Green   │ ███         │ ██          │ ████████    │ ███           │
│              │ Low         │ Instant     │ 2x Normal   │ Complex       │
├──────────────┼─────────────┼─────────────┼─────────────┼───────────────┤
│ CANARY ★     │ ██          │ ██          │ ███         │ ████          │
│ (This App)   │ Very Low    │ Fast        │ +10-20%     │ Advanced      │
└──────────────┴─────────────┴─────────────┴─────────────┴───────────────┘

★ Canary is ideal for: Production environments, high-traffic apps, 
  critical services, when automated rollback is required
```

---

## Voting App Canary Architecture

### Current Configuration

Your QA environment uses **Argo Rollouts** with NGINX Ingress for traffic splitting:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                VOTING APP QA - CANARY ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                     ┌─────────────────────┐                             │
│                     │    User Traffic     │                             │
│                     │ opsera-voting-app-  │                             │
│                     │ qa.agent.opsera.dev │                             │
│                     └──────────┬──────────┘                             │
│                                │                                        │
│                     ┌──────────▼──────────┐                             │
│                     │   NGINX Ingress     │                             │
│                     │   Controller        │                             │
│                     │ (Traffic Splitter)  │                             │
│                     └──────────┬──────────┘                             │
│                                │                                        │
│            ┌───────────────────┼───────────────────┐                    │
│            │ canary-weight: N% │                   │                    │
│            ▼                   ▼                   │                    │
│   ┌─────────────────┐   ┌─────────────────┐       │                    │
│   │  CANARY PODS    │   │  STABLE PODS    │       │                    │
│   │  (New Version)  │   │  (Old Version)  │       │                    │
│   │                 │   │                 │       │                    │
│   │  ┌───┐ ┌───┐    │   │  ┌───┐ ┌───┐    │       │                    │
│   │  │Pod│ │Pod│    │   │  │Pod│ │Pod│    │       │                    │
│   │  └───┘ └───┘    │   │  └───┘ └───┘    │       │                    │
│   │                 │   │  ┌───┐ ┌───┐    │       │                    │
│   │  N% of traffic  │   │  │Pod│ │Pod│    │       │                    │
│   │                 │   │  └───┘ └───┘    │       │                    │
│   │                 │   │ (100-N)% traffic│       │                    │
│   └────────┬────────┘   └────────┬────────┘       │                    │
│            │                     │                │                    │
│            └──────────┬──────────┘                │                    │
│                       │                           │                    │
│            ┌──────────▼──────────┐                │                    │
│            │   Argo Rollouts     │◄───────────────┘                    │
│            │   Controller        │   Monitors & Controls               │
│            │                     │                                     │
│            │ • Manages ReplicaSet│                                     │
│            │ • Updates Ingress   │                                     │
│            │ • Runs Analysis     │                                     │
│            │ • Triggers Rollback │                                     │
│            └─────────────────────┘                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Services Configuration

| Service | Type | Replicas | Canary Enabled |
|---------|------|----------|----------------|
| **vote** | Rollout | 4 | ✅ Yes |
| **result** | Rollout | 3 | ✅ Yes |
| **worker** | Deployment | 2 | ❌ No (background job) |
| **redis** | Deployment | 1 | ❌ No (stateless cache) |
| **db** | Deployment | 1 | ❌ No (stateful, separate strategy) |

### Traffic Steps Configuration

```yaml
# From: .opsera-voting-app/k8s/overlays/qa/patch-rollout-replicas.yaml
strategy:
  canary:
    steps:
    - setWeight: 10    # Step 1: 10% to canary
    - pause: {duration: 2m}  # Wait 2 minutes
    - setWeight: 30    # Step 2: 30% to canary
    - pause: {duration: 2m}  # Wait 2 minutes
    - setWeight: 60    # Step 3: 60% to canary
    - pause: {duration: 2m}  # Wait 2 minutes
    - setWeight: 100   # Step 4: Full promotion
```

---

## Step-by-Step Flow Simulation

### Scenario: Deploying Vote Service v2.0.0

Let's simulate deploying a new version of the vote service with a UI change.

```
┌─────────────────────────────────────────────────────────────────────────┐
│           CANARY DEPLOYMENT SIMULATION - VOTE SERVICE v2.0.0           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  INITIAL STATE (Before Deployment)                                     │
│  ════════════════════════════════                                      │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Stable Version: v1.0.0                                      │       │
│  │ Pods: 4/4 Running                                           │       │
│  │ Traffic: 100% → v1.0.0                                      │       │
│  │                                                             │       │
│  │   [Pod-1] [Pod-2] [Pod-3] [Pod-4]  ← All serving v1.0.0     │       │
│  │     ▲       ▲       ▲       ▲                               │       │
│  │     └───────┴───────┴───────┘                               │       │
│  │              100% Traffic                                   │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

                              │
                              │  CI/CD Pipeline pushes new image
                              │  Image: vote:20260127-abc123
                              │  Kustomize updates image tag
                              │  ArgoCD detects change and syncs
                              ▼

┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 0: CANARY CREATION (T+0:00)                                      │
│  ═══════════════════════════════                                       │
│                                                                         │
│  Argo Rollouts detects new image and creates canary ReplicaSet         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Stable: v1.0.0 (4 pods)          Canary: v2.0.0 (0 pods)    │       │
│  │                                                              │       │
│  │ [Pod-1] [Pod-2] [Pod-3] [Pod-4]     [Creating...]           │       │
│  │   100% Traffic                          0%                   │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Status: Progressing                                                   │
│  Message: "Starting canary rollout for vote-rollout"                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

                              │
                              │  Canary pod starts, passes health checks
                              ▼

┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 1: 10% TRAFFIC SHIFT (T+0:30)                                    │
│  ══════════════════════════════════                                    │
│                                                                         │
│  NGINX Ingress annotation updated: canary-weight: "10"                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Stable: v1.0.0 (4 pods)          Canary: v2.0.0 (1 pod)     │       │
│  │                                                              │       │
│  │ [Pod-1] [Pod-2] [Pod-3] [Pod-4]     [Canary-1]              │       │
│  │         90% Traffic                   10% Traffic            │       │
│  │             │                            │                   │       │
│  │             │  ┌─────────────────────────┤                   │       │
│  │             ▼  ▼                                             │       │
│  │  ┌─────────────────────────────────────────────────┐        │       │
│  │  │       ANALYSIS RUNNING (success-rate)          │        │       │
│  │  │  ┌─────┬─────┬─────┬─────┬─────┐               │        │       │
│  │  │  │ ✓   │ ✓   │ ✓   │ ✓   │ ✓   │  5/5 checks  │        │       │
│  │  │  │98.2%│97.5%│99.1%│98.8%│97.9%│  Passed!     │        │       │
│  │  │  └─────┴─────┴─────┴─────┴─────┘               │        │       │
│  │  │  Threshold: ≥95%  |  Actual: 98.3% avg        │        │       │
│  │  └─────────────────────────────────────────────────┘        │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Status: Progressing                                                   │
│  Step: 1/7 (setWeight: 10)                                             │
│  Analysis: Running (success-rate: 98.3%)                               │
│                                                                         │
│  ⏱️ Pause: 2 minutes remaining                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

                              │
                              │  2 minute pause completes
                              │  Analysis passed all checks
                              ▼

┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 2: 30% TRAFFIC SHIFT (T+2:30)                                    │
│  ══════════════════════════════════                                    │
│                                                                         │
│  NGINX Ingress annotation updated: canary-weight: "30"                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Stable: v1.0.0 (4 pods)          Canary: v2.0.0 (2 pods)    │       │
│  │                                                              │       │
│  │ [Pod-1] [Pod-2] [Pod-3] [Pod-4]  [Canary-1] [Canary-2]      │       │
│  │         70% Traffic                   30% Traffic            │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Traffic Distribution:                                                 │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │████████████████████████████████████████████████████████░░░░│        │
│  │◄──────────── Stable (70%) ────────────►│◄─ Canary (30%) ─►│        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  Status: Progressing                                                   │
│  Step: 3/7 (setWeight: 30)                                             │
│  ⏱️ Pause: 2 minutes remaining                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

                              │
                              │  Analysis continues passing
                              ▼

┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 3: 60% TRAFFIC SHIFT (T+4:30)                                    │
│  ══════════════════════════════════                                    │
│                                                                         │
│  NGINX Ingress annotation updated: canary-weight: "60"                 │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Stable: v1.0.0 (2 pods)          Canary: v2.0.0 (3 pods)    │       │
│  │                                                              │       │
│  │     [Pod-1] [Pod-2]         [Canary-1] [Canary-2] [Canary-3]│       │
│  │     40% Traffic                    60% Traffic               │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Traffic Distribution:                                                 │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│        │
│  │◄── Stable (40%) ──►│◄────────── Canary (60%) ─────────────►│        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  Status: Progressing                                                   │
│  Step: 5/7 (setWeight: 60)                                             │
│  ⏱️ Pause: 2 minutes remaining                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

                              │
                              │  Final analysis check passes
                              ▼

┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 4: 100% PROMOTION - COMPLETE! (T+6:30)                           │
│  ═══════════════════════════════════════════                           │
│                                                                         │
│  Canary is now the new Stable. Old ReplicaSet scaled to 0.             │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Old Stable: v1.0.0 (TERMINATED)  New Stable: v2.0.0         │       │
│  │                                                              │       │
│  │      (scaled to 0)         [Pod-1] [Pod-2] [Pod-3] [Pod-4]  │       │
│  │                                   100% Traffic               │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Traffic Distribution:                                                 │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│        │
│  │◄────────────────── v2.0.0 (100%) ─────────────────────────►│        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  ✅ Status: Healthy                                                     │
│  ✅ Promotion: Complete                                                 │
│  ✅ All users now on v2.0.0                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Timeline Summary

```
TIME      WEIGHT    ACTION                              ANALYSIS
────────────────────────────────────────────────────────────────────────
T+0:00    0%        Canary pod created                  -
T+0:30    10%       Traffic shifted                     Started
T+2:30    30%       Traffic increased                   Passed (98.3%)
T+4:30    60%       Traffic increased                   Passed (97.8%)
T+6:30    100%      Full promotion                      All Passed ✅
────────────────────────────────────────────────────────────────────────
Total Deployment Time: ~7 minutes
Users Affected by Issues: Maximum 60% (if failed at step 3)
```

---

## Analysis & Auto-Rollback

### Analysis Template Metrics

Your voting app uses three metrics for automated decision-making:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ANALYSIS METRICS CONFIGURATION                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  METRIC 1: SUCCESS RATE                                                │
│  ══════════════════════                                                │
│  Query: HTTP 2xx responses / Total requests                            │
│  Threshold: ≥ 95%                                                      │
│  Check Interval: 30 seconds                                            │
│  Samples: 5                                                            │
│  Failure Limit: 3 (rollback after 3 failures)                          │
│                                                                         │
│  Example:                                                              │
│  ┌─────┬─────┬─────┬─────┬─────┐                                       │
│  │96.2%│97.1%│95.8%│96.5%│97.0%│  ✅ PASS (all ≥95%)                   │
│  └─────┴─────┴─────┴─────┴─────┘                                       │
│                                                                         │
│  ┌─────┬─────┬─────┬─────┬─────┐                                       │
│  │96.2%│93.1%│91.8%│89.5%│88.0%│  ❌ FAIL (3+ below 95%)               │
│  └─────┴─────┴─────┴─────┴─────┘                                       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  METRIC 2: ERROR RATE                                                  │
│  ════════════════════                                                  │
│  Query: HTTP 5xx responses / Total requests                            │
│  Threshold: ≤ 5%                                                       │
│  Check Interval: 30 seconds                                            │
│  Samples: 5                                                            │
│  Failure Limit: 2 (rollback after 2 failures)                          │
│                                                                         │
│  Example:                                                              │
│  ┌─────┬─────┬─────┬─────┬─────┐                                       │
│  │ 1.2%│ 0.8%│ 1.5%│ 2.1%│ 1.8%│  ✅ PASS (all ≤5%)                    │
│  └─────┴─────┴─────┴─────┴─────┘                                       │
│                                                                         │
│  ┌─────┬─────┬─────┬─────┬─────┐                                       │
│  │ 1.2%│ 8.5%│12.3%│ 9.1%│ 7.8%│  ❌ FAIL (2+ above 5%)                │
│  └─────┴─────┴─────┴─────┴─────┘                                       │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  METRIC 3: P99 LATENCY                                                 │
│  ═════════════════════                                                 │
│  Query: 99th percentile response time                                  │
│  Threshold: ≤ 500ms                                                    │
│  Check Interval: 30 seconds                                            │
│  Samples: 5                                                            │
│  Failure Limit: 2 (rollback after 2 failures)                          │
│                                                                         │
│  Example:                                                              │
│  ┌──────┬──────┬──────┬──────┬──────┐                                  │
│  │ 245ms│ 312ms│ 289ms│ 356ms│ 298ms│  ✅ PASS (all ≤500ms)            │
│  └──────┴──────┴──────┴──────┴──────┘                                  │
│                                                                         │
│  ┌──────┬──────┬──────┬──────┬──────┐                                  │
│  │ 245ms│ 612ms│ 789ms│ 856ms│ 923ms│  ❌ FAIL (2+ above 500ms)        │
│  └──────┴──────┴──────┴──────┴──────┘                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Auto-Rollback Scenario Simulation

```
┌─────────────────────────────────────────────────────────────────────────┐
│               AUTO-ROLLBACK SIMULATION - BAD DEPLOYMENT                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STEP 1: 10% Traffic (T+0:30)                                          │
│  ─────────────────────────────                                         │
│  Analysis Check 1: Success Rate = 98.2%  ✅                             │
│  Analysis Check 2: Success Rate = 97.5%  ✅                             │
│  Analysis Check 3: Success Rate = 96.1%  ✅                             │
│  Analysis Check 4: Success Rate = 95.8%  ✅                             │
│  Analysis Check 5: Success Rate = 97.0%  ✅                             │
│  Result: PASSED → Proceed to 30%                                       │
│                                                                         │
│  STEP 2: 30% Traffic (T+2:30)                                          │
│  ─────────────────────────────                                         │
│  Analysis Check 1: Success Rate = 94.2%  ⚠️  (below 95%)                │
│  Analysis Check 2: Success Rate = 91.5%  ❌ (below 95%)                 │
│  Analysis Check 3: Success Rate = 88.1%  ❌ (below 95%)                 │
│  Analysis Check 4: Success Rate = 85.8%  ❌ (below 95%)  FAILURE LIMIT! │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  ⚠️  ANALYSIS FAILED - INITIATING AUTOMATIC ROLLBACK        │       │
│  │                                                             │       │
│  │  Reason: success-rate metric failed 3 consecutive times    │       │
│  │  Action: Rolling back to stable version v1.0.0             │       │
│  │  Impact: Only 30% of users experienced degraded service    │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ROLLBACK EXECUTION (T+3:30)                                           │
│  ───────────────────────────                                           │
│  1. NGINX annotation: canary-weight: "0"                               │
│  2. All traffic routed to stable pods                                  │
│  3. Canary ReplicaSet scaled to 0                                      │
│  4. Rollout status: Degraded                                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Final State:                                                │       │
│  │                                                             │       │
│  │ Stable: v1.0.0 (4 pods)          Canary: v2.0.0 (0 pods)   │       │
│  │ [Pod-1] [Pod-2] [Pod-3] [Pod-4]      (terminated)          │       │
│  │         100% Traffic                     0%                 │       │
│  │                                                             │       │
│  │ ✅ Service restored - all users on stable v1.0.0            │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Manual Operations

### Common Commands

```bash
# ═══════════════════════════════════════════════════════════════════════
#                    ARGO ROLLOUTS CLI COMMANDS
# ═══════════════════════════════════════════════════════════════════════

# View rollout status (real-time)
kubectl argo rollouts get rollout vote-rollout -n voting-app-qa -w

# View rollout status (one-time)
kubectl argo rollouts status vote-rollout -n voting-app-qa

# List all rollouts
kubectl argo rollouts list rollouts -n voting-app-qa

# ═══════════════════════════════════════════════════════════════════════
#                       MANUAL INTERVENTIONS
# ═══════════════════════════════════════════════════════════════════════

# PROMOTE: Skip remaining steps and promote canary to stable
kubectl argo rollouts promote vote-rollout -n voting-app-qa

# FULL PROMOTE: Promote without waiting for remaining analysis
kubectl argo rollouts promote vote-rollout -n voting-app-qa --full

# ABORT: Stop rollout and rollback to stable
kubectl argo rollouts abort vote-rollout -n voting-app-qa

# RETRY: Retry a failed/aborted rollout
kubectl argo rollouts retry rollout vote-rollout -n voting-app-qa

# PAUSE: Pause at current step
kubectl argo rollouts pause vote-rollout -n voting-app-qa

# RESUME: Resume a paused rollout
kubectl argo rollouts resume vote-rollout -n voting-app-qa

# ═══════════════════════════════════════════════════════════════════════
#                      SET SPECIFIC TRAFFIC WEIGHT
# ═══════════════════════════════════════════════════════════════════════

# Skip to specific traffic weight (e.g., jump to 50%)
kubectl argo rollouts set weight vote-rollout 50 -n voting-app-qa

# ═══════════════════════════════════════════════════════════════════════
#                        VIEW ANALYSIS RUNS
# ═══════════════════════════════════════════════════════════════════════

# List analysis runs for a rollout
kubectl get analysisrun -n voting-app-qa -l rollout=vote-rollout

# View analysis run details
kubectl describe analysisrun <analysis-run-name> -n voting-app-qa
```

### Decision Flowchart

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CANARY ROLLOUT DECISION GUIDE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Is the rollout stuck/paused?                                          │
│         │                                                              │
│         ├── YES ──► Is it a scheduled pause?                           │
│         │                 │                                            │
│         │                 ├── YES ──► Wait or run: kubectl argo       │
│         │                 │           rollouts resume <rollout>        │
│         │                 │                                            │
│         │                 └── NO ──► Check analysis status:            │
│         │                            kubectl argo rollouts get <name>  │
│         │                                                              │
│         └── NO ──► Is analysis failing?                                │
│                          │                                             │
│                          ├── YES ──► Is the failure legitimate?        │
│                          │                 │                           │
│                          │                 ├── YES ──► Fix code, push  │
│                          │                 │           new version     │
│                          │                 │                           │
│                          │                 └── NO ──► False positive?  │
│                          │                            Run: kubectl argo│
│                          │                            rollouts promote │
│                          │                                             │
│                          └── NO ──► Rollout healthy, let it proceed   │
│                                                                         │
│  Need emergency rollback?                                              │
│         │                                                              │
│         └── YES ──► kubectl argo rollouts abort vote-rollout           │
│                     -n voting-app-qa                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Monitoring & Observability

### Key Metrics to Watch

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY DASHBOARD METRICS                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TRAFFIC METRICS (Compare Stable vs Canary)                            │
│  ──────────────────────────────────────────                            │
│                                                                         │
│  Request Rate                    Error Rate                            │
│  ┌────────────────────────┐     ┌────────────────────────┐             │
│  │   ╭──────╮             │     │                        │             │
│  │  ╱       ╲    Stable   │     │  ▁▁▁▁▁▁▁▁▁▁▁▁  Stable │             │
│  │ ╱         ╲───────────▶│     │                        │             │
│  │╱           ╲           │     │  ████████████  Canary  │             │
│  │             ╲  Canary  │     │                        │             │
│  └────────────────────────┘     └────────────────────────┘             │
│  Good: Similar patterns         Alert: Canary errors higher            │
│                                                                         │
│  Response Time (P99)            Success Rate                           │
│  ┌────────────────────────┐     ┌────────────────────────┐             │
│  │                        │     │  ████████████████████  │             │
│  │  ▁▁▁▁▁▁▁▁▁▁▁▁  Stable │     │  Stable: 99.2%         │             │
│  │  ▁▁▁▁▁▁▁▁▁▁▁▁  Canary │     │  ████████████████████  │             │
│  │                        │     │  Canary: 98.8%         │             │
│  └────────────────────────┘     └────────────────────────┘             │
│  Good: Both under 500ms         Good: Both above 95%                   │
│                                                                         │
│  POD METRICS                                                           │
│  ───────────────                                                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Pod        │ CPU    │ Memory │ Restarts │ Status  │ Version │       │
│  ├─────────────────────────────────────────────────────────────┤       │
│  │ vote-stable-1  │ 45m  │ 128Mi │    0     │ Running │ v1.0.0 │       │
│  │ vote-stable-2  │ 52m  │ 132Mi │    0     │ Running │ v1.0.0 │       │
│  │ vote-canary-1  │ 48m  │ 125Mi │    0     │ Running │ v2.0.0 │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Prometheus Queries

```promql
# ═══════════════════════════════════════════════════════════════════════
#                     PROMETHEUS QUERIES FOR CANARY
# ═══════════════════════════════════════════════════════════════════════

# Success Rate (per revision)
sum(rate(nginx_ingress_controller_requests{
  service="vote",
  status=~"2.*"
}[5m])) by (canary)
/
sum(rate(nginx_ingress_controller_requests{
  service="vote"
}[5m])) by (canary)

# Request Rate Comparison
sum(rate(nginx_ingress_controller_requests{service="vote"}[5m])) by (canary)

# P99 Latency Comparison
histogram_quantile(0.99,
  sum(rate(nginx_ingress_controller_request_duration_seconds_bucket{
    service="vote"
  }[5m])) by (le, canary)
)

# Error Rate by Version
sum(rate(nginx_ingress_controller_requests{
  service="vote",
  status=~"5.*"
}[5m])) by (canary)
/
sum(rate(nginx_ingress_controller_requests{service="vote"}[5m])) by (canary)
```

---

## Troubleshooting Guide

### Common Issues and Solutions

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TROUBLESHOOTING QUICK REFERENCE                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ISSUE: Rollout stuck at "Progressing"                                 │
│  ─────────────────────────────────────                                 │
│  Symptoms: Rollout doesn't advance past a step                         │
│  Causes:                                                               │
│    1. Canary pods failing health checks                                │
│    2. Analysis template returning errors                               │
│    3. Prometheus not accessible                                        │
│  Debug:                                                                │
│    kubectl argo rollouts get vote-rollout -n voting-app-qa            │
│    kubectl describe pod -l app=vote -n voting-app-qa                  │
│    kubectl logs -l app=vote -n voting-app-qa --tail=50                │
│  Fix:                                                                  │
│    - Check pod events for errors                                       │
│    - Verify Prometheus connectivity                                    │
│    - Review analysis run: kubectl get analysisrun -n voting-app-qa    │
│                                                                         │
│  ISSUE: Analysis always failing                                        │
│  ──────────────────────────────                                        │
│  Symptoms: Rollout aborts immediately after analysis starts            │
│  Causes:                                                               │
│    1. Prometheus not deployed                                          │
│    2. Wrong service name in query                                      │
│    3. No traffic to generate metrics                                   │
│  Debug:                                                                │
│    kubectl get analysisrun -n voting-app-qa -o yaml                   │
│    # Check "message" field for actual error                            │
│  Fix:                                                                  │
│    - Deploy Prometheus if missing                                      │
│    - Verify query returns data in Prometheus UI                        │
│    - Generate traffic with: curl -s https://voting-app-qa...          │
│                                                                         │
│  ISSUE: Traffic not shifting                                           │
│  ───────────────────────────                                           │
│  Symptoms: canary-weight annotation not updating                       │
│  Causes:                                                               │
│    1. NGINX ingress controller not configured                          │
│    2. Ingress name mismatch in rollout config                          │
│    3. Missing canary annotations on ingress                            │
│  Debug:                                                                │
│    kubectl get ingress voting-app -n voting-app-qa -o yaml            │
│    # Check nginx.ingress.kubernetes.io/canary-weight value             │
│  Fix:                                                                  │
│    - Verify ingress has canary: "true" annotation                      │
│    - Ensure stableIngress name matches actual ingress name             │
│                                                                         │
│  ISSUE: Rollout degraded after abort                                   │
│  ────────────────────────────────                                      │
│  Symptoms: Status shows "Degraded", won't accept new updates           │
│  Cause: Previous rollout was aborted and needs retry                   │
│  Fix:                                                                  │
│    kubectl argo rollouts retry rollout vote-rollout -n voting-app-qa  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

### Your QA Environment Canary Configuration

| Setting | Value |
|---------|-------|
| **Strategy** | Canary with NGINX traffic splitting |
| **Steps** | 10% → 30% → 60% → 100% |
| **Pause Duration** | 2 minutes between steps |
| **Total Rollout Time** | ~7 minutes (if all passes) |
| **Success Rate Threshold** | ≥ 95% |
| **Error Rate Threshold** | ≤ 5% |
| **P99 Latency Threshold** | ≤ 500ms |
| **Auto-Rollback** | Yes, on 3 consecutive failures |

### Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CANARY QUICK REFERENCE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VIEW STATUS    kubectl argo rollouts get vote-rollout -n voting-app-qa│
│  PROMOTE        kubectl argo rollouts promote vote-rollout -n voting-..|
│  ABORT          kubectl argo rollouts abort vote-rollout -n voting-app-qa│
│  RETRY          kubectl argo rollouts retry rollout vote-rollout -n ...|
│                                                                         │
│  QA VOTE URL    https://opsera-voting-app-qa.agent.opsera.dev          │
│  QA RESULT URL  https://results.opsera-voting-app-qa.agent.opsera.dev  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*Document generated for Voting App Enterprise - QA Environment*
*Powered by Opsera Unified DevOps Platform*
