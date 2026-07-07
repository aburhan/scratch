# EKS to GKE migration readiness report

**Run:** `run-001` **Generated:** `2026-07-07T17:32:38.100295Z`

## 1. Executive summary

- **Overall Grade:** `A-` — The environment is highly ready for
  migration with zero blocker items.
- **Total Workloads:** 80 workloads analyzed (70 Ready,
  10 Tractable, 0 High-risk, 0 Blocked).
- **Single Most Important Blocker:** None. All platform configurations
  are standard and Autopilot compatible.
- **Estimated Effort:** 55.0 to 82.5 engineer-days (P50/P90 range).
- **Recommended First Move:** Establish target GKE landing zone and deploy
  the stateless tractable workloads (`fat-shard`, `spread-tiny`).
- **Biggest Risk:** Data migration synchronization lag for the RDS database.

| Cluster | Grade | Workloads (R/T/H/B) | Effort (P50/P90) |
| ----------- | --------- | ------------------- | ---------------- |
| migratorium-eks | A- | 70/10/0/0 | 55.0/82.5 days |

## 2. Scope

Discovered 1 cluster across 1 region (`us-east-1`).
- **Cluster:** `migratorium-eks` (Version: `1.33`)
- **Nodes:** 4 nodes across 2 nodegroups.
- **Namespaces in scope:** kube-system, migratorium

## 3. Scorecard

### Workload Bucket Distribution

- **Ready:** 70 (87.5%)
- **Tractable:** 10 (12.5%)
- **High-risk:** 0 (0.0%)
- **Blocked:** 0 (0.0%)

### Detailed Scorecard Table

| Namespace | Workload Name | Kind | Statelessness | Identity | Network | Data | Composite Score | Bucket |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| kube-system | aws-load-balancer-controller | Deployment | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| kube-system | coredns | Deployment | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| kube-system | ebs-csi-controller | Deployment | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| kube-system | efs-csi-controller | Deployment | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| kube-system | aws-node | DaemonSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| kube-system | ebs-csi-node | DaemonSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| kube-system | ebs-csi-node-windows | DaemonSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| kube-system | efs-csi-node | DaemonSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| kube-system | kube-proxy | DaemonSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| kube-system | aws-load-balancer-controller-85bdbb58b6 | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| kube-system | coredns-6b7557d946 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| kube-system | ebs-csi-controller-54994b5f64 | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| kube-system | efs-csi-controller-8d4ddd846 | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | fat-shard | Deployment | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen | Deployment | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | media-svc | Deployment | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | metrics-agg | Deployment | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | migration-ui | Deployment | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | report-operator | Deployment | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | spread-tiny | Deployment | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | status-collector | Deployment | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | event-store | StatefulSet | 2/3 | 3/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | event-store-snapshot | CronJob | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | event-store-seed | Job | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | event-store-snapshot-29720340 | Job | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | event-store-snapshot-29721780 | Job | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | event-store-snapshot-29723220 | Job | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | rds-seed | Job | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | fat-shard-5cbf85488d | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-57cbfd5ccf | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-58d5566857 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-5b5cff8f64 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-5f998c7795 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-6459487bdc | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-655655c475 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-88dcb7d8d | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-b74487f99 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | loadgen-f9bcb475d | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | media-svc-56446dccd5 | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-5c5d4664dd | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-65d965b58 | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-6655bb99b8 | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-6bb5d7ddf6 | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-75566979dc | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-784dd5df7d | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-785c87dd9c | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | media-svc-86d77df7df | ReplicaSet | 2/3 | 2/3 | 2/3 | 3/3 | 0.75 | **Ready** |
| migratorium | metrics-agg-5997b69b54 | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | metrics-agg-648569f574 | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | metrics-agg-6b8898dc4d | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | metrics-agg-6dbbc9ccdd | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | metrics-agg-7958f5b5f8 | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | metrics-agg-dd67687c | ReplicaSet | 3/3 | 2/3 | 2/3 | 3/3 | 0.83 | **Ready** |
| migratorium | migration-ui-546695d64d | ReplicaSet | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | migration-ui-554886ddfb | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | migration-ui-57995567c6 | ReplicaSet | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | migration-ui-5c4f5d8c6f | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | migration-ui-66f6d4bc4b | ReplicaSet | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | migration-ui-68564559 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | migration-ui-6f7d69568d | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | migration-ui-77f59c7748 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | migration-ui-7d8b944b94 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | migration-ui-86665d4f45 | ReplicaSet | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | migration-ui-9bd5c5748 | ReplicaSet | 3/3 | 3/3 | 2/3 | 2/3 | 0.83 | **Ready** |
| migratorium | report-operator-558c4cfcfd | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | report-operator-5b58db4b7b | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | report-operator-7c9d7fffff | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | report-operator-7cd65c5dcb | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | report-operator-7d9b479bf8 | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | report-operator-86546dbd7c | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | spread-tiny-5f6777659c | ReplicaSet | 3/3 | 3/3 | 2/3 | 3/3 | 0.92 | **Ready** |
| migratorium | status-collector-69f5f9bbb4 | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-6cd8cfdf7b | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-6f88bddc7c | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-756dfd7db | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-75c99994d5 | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-768955b947 | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-895c5779c | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-b999ffdcd | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |
| migratorium | status-collector-cf757765 | ReplicaSet | 3/3 | 2/3 | 2/3 | 1/3 | 0.67 | **Tractable** |

## 4. Blockers

Zero active blockers were identified in the discovery artifacts.
All workloads are GKE Autopilot compatible.

## 5. Phased plan

Fitted plan for the 60-day migration window:

| Phase | Stage | Description | Duration | Target Schedule |
| :--- | :--- | :--- | :--- | :--- |
| Phase 1 | Foundation | Landing zone, VPC setup, GKE Dataplane V2 setup, ECR mirroring. | 10 days | Day 1 to Day 10 |
| Phase 2 | Tier-2 wave | Migration of stateless Ready workloads (`fat-shard`, `spread-tiny`). | 15 days | Day 11 to Day 25 |
| Phase 3 | Stateful + Tier-1 | Stateful workloads and database replication setup (`media-svc`, `event-store`). | 20 days | Day 26 to Day 45 |
| Phase 4 | Tier-0 | Final database sync, DNS cutover, and traffic routing. | 10 days | Day 46 to Day 55 |
| Phase 5 | Decommission | Soak period, EKS teardown, and FinOps cleanup. | 5 days | Day 56 to Day 60 |

## 6. Effort estimate

- **P50 Estimate:** 55.0 engineer-days.
- **P90 Estimate:** 82.5 engineer-days.
- **Assumptions:** Organization coefficient is 1.0 (standard baseline experience).

## 7. Risks and guardrails

- **Risk 1:** homogeneous database sync lag for `rds:migratorium-eks-status`
  cutover.
  - *Guardrail*: Database replication sync lag SLO of < 30 seconds before
    cutover execution.
- **Risk 2:** Cross-cloud data egress fees during replication.
  - *Guardrail*: Daily cost monitoring with budget alerts capped at $5,000
    for the migration period.

## 8. Open questions

1. Do we want to migrate the `kube-system` platform components (like EBS/EFS
   CSI driver config) or rely purely on GKE built-in CSI storage drivers?
   (Recommended: GKE built-in CSI drivers).
2. Confirm the expected availability SLO during the final cutover window
   (planned downtime vs zero-downtime cutover).
