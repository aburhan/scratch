# Discovery Validation Report

## Validation Checks

- **Check 1: Every namespace has a workload count** -> **PASS**
  - Namespace `default`: 0 workloads
  - Namespace `kube-node-lease`: 0 workloads
  - Namespace `kube-system`: 13 workloads
  - Namespace `kube-public`: 0 workloads
  - Namespace `migratorium`: 67 workloads

- **Check 2: Every IRSA ServiceAccount maps to a real IAM role** -> **PASS**
  - `kube-system/aws-load-balancer-controller`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-lb-controller-20260529033523866100000001`
    Status: Exists
  - `kube-system/ebs-csi-controller-sa`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-ebs-csi-2026052901384852730000000f`
    Status: Exists
  - `kube-system/efs-csi-controller-sa`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-efs-csi-20260529013848527400000010`
    Status: Exists
  - `kube-system/efs-csi-node-sa`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-efs-csi-20260529013848527400000010`
    Status: Exists
  - `migratorium/media-svc`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-media-svc-2026052901384852720000000e`
    Status: Exists
  - `migratorium/metrics-agg`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-metrics-agg-2026052901384852460000000d`
    Status: Exists
  - `migratorium/snapshot-sa`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-status-collector-20260529013848527500000011`
    Status: Exists
  - `migratorium/status-collector`
    -> `arn:aws:iam::162908504356:role/migratorium-eks-status-collector-20260529013848527500000011`
    Status: Exists

- **Check 3: Every PV resolves to a real EBS or EFS resource** -> **PASS**
  - PV `media-efs-pv`
    -> EFS Volume `fs-094d064644a3ee6b8`
  - PV `pvc-f6329c07-2b2d-4e6b-8405-085f39dc6d9d`
    -> EBS Volume `vol-0c59e3c4e2179815f`

## Validation Summary

Status: **SUCCESS** (0 FAILs)
