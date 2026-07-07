# EKS Discovery Inventory Summary

Captured at: `2026-07-07T17:31:51.752912Z`  
Source Account: `162908504356`  
Region: `us-east-1`

## Estate Overview

- **Total Clusters:** 1
- **Total Nodes:** 4
- **Total IRSA Bindings:** 8
- **Total Secrets:** 2
- **Escalations/Risks:** 0

## Clusters

### migratorium-eks
- **K8s Version:** `1.33` (Platform: `eks.39`)
- **Auth Mode:** `API_AND_CONFIG_MAP`
- **VPC:** `vpc-0907eb4d2c8977ffc`
- **CNI Plugin:** `vpc-cni`
- **Endpoints Access:** Public: `True`, Private: `True`

#### Node Pools

- **Pool Name:** `general-20260529025442419400000005`
  - Instance Types: `m5.xlarge`
  - Desired Size: 3 (Min: 1, Max: 3)
  - Capacity Type: `ON_DEMAND`
  - AMI Type: `AL2023_x86_64_STANDARD`
- **Pool Name:** `dedicated-20260529025442419400000003`
  - Instance Types: `m5.large`
  - Desired Size: 1 (Min: 1, Max: 2)
  - Capacity Type: `ON_DEMAND`
  - AMI Type: `AL2023_x86_64_STANDARD`

#### Key Risks / Escalations

None identified.
