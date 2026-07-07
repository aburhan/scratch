# EKS to GKE migration readiness report

**Run:** {{run_id}} **Generated:** {{date}}

## 1. Executive summary

Five bullets. Overall grade. The single most important blocker. The effort
range. The recommended first move. The biggest risk.

| Cluster     | Grade     | Workloads (R/T/H/B) | Effort (P50/P90) |
| ----------- | --------- | ------------------- | ---------------- |
| {{cluster}} | {{grade}} | {{counts}}          | {{effort}}       |

## 2. Scope

What was discovered: clusters, namespaces, workloads, regions, accounts.
Reference the inventory.

## 3. Scorecard

Per cluster, per workload class. Bucket distribution. The 4-axis breakdown for
any workload scoring below Tractable.

## 4. Blockers

Named, categorized, each with a resolution path. Cross-reference blockers.md.

## 5. Phased plan

Foundation → Tier-2 wave → Stateful+Tier-1 → Tier-0 → Decommission. Gantt-shaped
table fitted to the stated window.

## 6. Effort estimate

P50/P90 range. State the org-coefficient assumption. Show the driver math.

## 7. Risks and guardrails

Each named risk paired with a measurable guardrail.

## 8. Open questions

Decisions the user must make before the migration proceeds.
