# Upstream Data Dependencies Mapping

This document maps Kubernetes workloads to their discovered external and
internal data sources.

## Namespace `migratorium`

- **Workload:** `media-svc` (Deployment)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `migration-ui` (Deployment)
  - RDS: `migratorium-eks-status`
- **Workload:** `status-collector` (Deployment)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
  - RDS: `migratorium-eks-status`
- **Workload:** `rds-seed` (Job)
  - RDS: `migratorium-eks-status`
- **Workload:** `media-svc-56446dccd5` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-5c5d4664dd` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-65d965b58` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-6655bb99b8` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-6bb5d7ddf6` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-75566979dc` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-784dd5df7d` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-785c87dd9c` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `media-svc-86d77df7df` (ReplicaSet)
  - EFS: `fs-094d064644a3ee6b8`
- **Workload:** `migration-ui-546695d64d` (ReplicaSet)
  - RDS: `migratorium-eks-status`
- **Workload:** `migration-ui-57995567c6` (ReplicaSet)
  - RDS: `migratorium-eks-status`
- **Workload:** `migration-ui-66f6d4bc4b` (ReplicaSet)
  - RDS: `migratorium-eks-status`
- **Workload:** `migration-ui-86665d4f45` (ReplicaSet)
  - RDS: `migratorium-eks-status`
- **Workload:** `migration-ui-9bd5c5748` (ReplicaSet)
  - RDS: `migratorium-eks-status`
- **Workload:** `status-collector-69f5f9bbb4` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
- **Workload:** `status-collector-6cd8cfdf7b` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
  - RDS: `migratorium-eks-status`
- **Workload:** `status-collector-6f88bddc7c` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
- **Workload:** `status-collector-756dfd7db` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
  - RDS: `migratorium-eks-status`
- **Workload:** `status-collector-75c99994d5` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
- **Workload:** `status-collector-768955b947` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
- **Workload:** `status-collector-895c5779c` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
- **Workload:** `status-collector-b999ffdcd` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`
  - RDS: `migratorium-eks-status`
- **Workload:** `status-collector-cf757765` (ReplicaSet)
  - SECRETSMANAGER: `migratorium-eks/signing-key`
  - S3: `migratorium-eks-config-demo`

