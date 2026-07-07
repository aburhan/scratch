---
name: migration-assessment
description: Score an Amazon Elastic Kubernetes Service (EKS) estate's readiness for migration to Google Kubernetes Engine (GKE) and produce a graded readiness report with named blockers, workload risk ratings, and a phased plan.
---

# Migration assessment

<!--
  Disabling markdownlint MD029 to provide explicit ordering to avoid
  confusing the LLM.
-->
<!-- markdownlint-disable MD029 -->

You are a migration architect. Read the discovery inventory and produce a
graded, opinionated readiness report. Your output represents the document a
migration consultant would write after week one.

## Purpose

Convert the raw inventory into a decision-ready document. This document includes
a scorecard, blockers list, per-workload risk ratings, a phased plan, and an
effort estimate.

## When to use this skill

- Immediately after executing `eks-discovery`.
- When the user asks "are we ready to migrate?" or "what is blocking the
  migration?".
- When the user wants an effort estimate before committing budget.
- To produce a board-ready summary slide of migration risk.

## Prerequisites

- The following files from the discovery phase must exist in the input
  directory:
    - `01-discovery/inventory.json`
    - `01-discovery/irsa-map.tsv`
    - `01-discovery/inventory-summary.md`
    - `01-discovery/workload-handoff/` (directory containing per-namespace
      workload data)
- The orchestrator's `00-orchestrator-state.json` contains the user's
  constraints. These constraints include compliance, Service Level Objectives
  (SLOs), window, and cost ceiling.
- If the executing agent does not find these prerequisites, it must halt
  execution and report the missing requirements to the user.

## Procedure

### Step 1 — Load inputs

Read `01-discovery/inventory.json`, the namespace workload files under
`01-discovery/workload-handoff/`, and `00-orchestrator-state.json`. Build an
in-memory model: clusters → namespaces → workloads → dependencies.

### Step 2 — Score each workload

Score every workload in the namespace workload files under
`01-discovery/workload-handoff/` on four axes from 0 to 3, where 3 represents
the lowest risk:

- **Stateless-ness:** Assess whether the workload is stateless, has
  PersistentVolumeClaim (PVC) storage (ReadWriteOnce vs. ReadWriteMany (RWX)),
  or uses local hostPath.
- **Identity portability:** Evaluate IAM Roles for Service Accounts (IRSA)
  configurations and potential Identity and Access Management (IAM) gaps when
  mapping to Google Cloud IAM.
- **Network portability:** Evaluate service and ingress configurations, such as
  ClusterIP, LoadBalancer, Application Load Balancer (ALB), Network Load
  Balancer (NLB), or Virtual Private Cloud (VPC) Container Network Interface
  (CNI) features.
- **Data dependencies:** Assess homogeneous vs. heterogeneous data sinks (such
  as DynamoDB, Amazon Relational Database Service (RDS), Aurora) and
  cross-account access.

Composite score = sum / 12. Buckets:

- **Ready** (>= 0.75): low risk, fits standard pattern.
- **Tractable** (0.50–0.74): translatable with explicit plan, no rewrite.
- **High-risk** (0.25–0.49): plan + dry-run + extra validation. Consider
  re-platforming sub-components.
- **Blocked** (< 0.25): The agent cannot migrate these workloads as-is. They
  require design work or must remain on Amazon Elastic Kubernetes Service (EKS)
  or re-platform.

### Step 3 — Cluster-level scoring

Aggregate workload scores per cluster, plus add cluster-level signals:

- Control plane version drift from the latest GKE-supported version. For
  example, EKS 1.27 has no drift because GKE supports up to 1.31. EKS 1.24 has
  drift. Plan a Kubernetes version bump pre- or post-migration.
- Number of custom CustomResourceDefinitions (CRDs) / operators in use (more
  equals more translation work).
- Number of platform-level DaemonSets (more equals more parallel translation).
- Whether the cluster authentication mode is `CONFIG_MAP` (legacy). This
  requires `aws-auth` translation to GKE IAM.
- Whether the cluster enables secrets-at-rest encryption and which Key
  Management Service (KMS) key it uses. Google Kubernetes Engine (GKE) parity
  requires this configuration.

### Step 4 — Identify blockers

Walk the cluster-level escalations in `01-discovery/inventory.json` and the
workload scoring. Identify active blockers, such as a custom Container Network
Interface (CNI), hostNetwork settings, non-Amazon Elastic File System (EFS)
ReadWriteMany (RWX) PersistentVolumeClaims (PVCs), Karpenter NodePools, App
Mesh, cross-account Amazon Elastic Container Registry (ECR) pulls, federated
Single Sign-On (SSO), or compliance scope. Propose appropriate resolution paths
based on Google Cloud architecture standards.

### Step 5 — Effort estimate

For each cluster, estimate the migration effort in engineer-days by analyzing
the complexity of the workloads (Ready vs. Tractable vs. High-risk), ingress,
IAM roles, storage, and database configurations.

Multiply the total effort by an organizational coefficient. Use 0.8 if the team
has prior GKE experience, or 1.5 if they have no Google Cloud experience.
Express the final estimate as a range (P50 / P90).

### Step 6 — Phased plan

Produce a recommended phasing:

1.  **Foundation**: landing zone, identity, network plumbing, observability
    stack, registry mirroring.
2.  **Tier-2 first wave**: stateless tractable workloads. Build the muscle.
3.  **Stateful + tier-1**: one workload at a time, with full validation.
4.  **Tier-0**: final, slowest cutovers, extended soak.
5.  **Decommission**: EKS teardown, post-mortem, FinOps tune.

Adjust to the user's stated window. If the user has a 30-day window, collapse
the phases and tighten the risks. Surface this situation to the user.

### Step 7 — Risks and guardrails

For each named risk, propose a guardrail:

- "RDS to Cloud SQL homogeneous move replication lag during cutover" →
  guardrail: replication-lag SLO, pause if > 30s for 5 minutes.
- "Cross-cloud egress cost during co-existence" → guardrail: budget alert,
  traffic dashboard, capped co-existence period.
- "Workload Identity (WI) binding drift" → guardrail: Continuous Integration
  (CI) check that compares IRSA to WI map to daily live state.
- "GKE cluster cost overrun versus EKS baseline" → guardrail: monthly FinOps
  review and an auto-scaler tuning playbook.

### Step 8 — Compose the report

Render `02-assessment/readiness-report.md` from the readiness-report template
(`assets/readiness-report-template.md`). Sections, in order:

1.  Executive summary (1 page, 5 bullets, 1 chart-able scorecard table).
2.  Scope (what was discovered).
3.  Scorecard (per cluster, per workload class).
4.  Blockers (named, categorized, with resolution paths).
5.  Phased plan (Gantt-shaped table).
6.  Effort estimate (P50 / P90 range, org-coefficient assumption stated).
7.  Risks & guardrails.
8.  Open questions for the user.

Also render `02-assessment/blockers.md` from `assets/blockers-template.md` — a
self-contained checklist of every blocker with owner and target close date.

## Decision points

- **What counts as tier-0?** Default to whatever the user said in
  `00-orchestrator-state.json` if not stated, infer by SLO and revenue impact
  and surface for confirmation.
- **Engine change vs lift-and-shift for data.** Default to homogeneous moves
  (RDS to Cloud SQL same engine). Engine change is escalation-only.
- **GKE Autopilot vs Standard at the cluster level.** Default to Autopilot for
  clusters where more than 80% of workloads are stateless and fit within
  Autopilot's resource constraints. Otherwise, recommend Standard. Explicitly
  recommend the cluster mode in the report.

## Outputs and deliverables

```text
02-assessment/
├── readiness-report.md   # The headline document.
└── blockers.md           # Checklist of every blocker.
```

## Validation

- The readiness report evaluates and reflects every workload in the workload
  files under `01-discovery/workload-handoff/`.
- The blockers list provides a resolution path for every blocker, leaving none
  as "TBD".
- Verify that the effort estimate range is sane. Ensure there are no $0
  estimates and no estimates exceeding two years without explicit reasoning.
- Verify that the phased plan fits within the user's stated window. If it does
  not, the report must state this clearly on page 1.

## Escalation triggers

- The user's stated window is impossible at the P90 estimate. Escalate before
  continuing.
- Compliance scope includes a regime where GKE has a different feature set than
  EKS for an in-scope workload. Surface explicitly.
- Escalate if the effort estimate shows more than 50% of workloads in the
  "High-risk" or "Blocked" buckets. Recommend that the user revisit the scope.

## Common pitfalls

- **Hand-waving the effort estimate.** Don't say "a few weeks". Use the
  per-driver cost table. Defend your number.
- **Treating IRSA as a 1:1 map to Workload Identity.** It usually is a 1:1 map,
  but the few exceptions are critical. Score each role individually.
- **Underestimating data moves.** Heterogeneous data store changes are projects
  of their own. If the user wants to migrate the Kubernetes fleet but keep
  Amazon Relational Database Service (RDS), that is a valid choice. Surface this
  option clearly to the user.
- **Listing risks without guardrails.** A risk without a guardrail is a guess.
  Every risk gets a measurable guardrail.
