# Migration blockers checklist

<!-- markdownlint-disable MD060 -->

This checklist tracks all active blockers preventing migration from Amazon
Elastic Kubernetes Service (EKS) to Google Kubernetes Engine (GKE). All blockers
must be resolved or have an approved mitigation plan before proceeding with
migration execution.

## Active blockers

| Status | Blocker ID | Category   | Target / Workload | Description                                                                                                          | Resolution Path                                                             | Owner | Target Date |
| :----: | :--------- | :--------- | :---------------- | :------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------- | :---- | :---------- |
|  [ ]   | B-001      | CNI        | Cluster           | Custom Container Network Interface (CNI) (non-Virtual Private Cloud (VPC) CNI)                                       | Migrate to GKE Dataplane V2 or Cilium                                       |       |             |
|  [ ]   | B-002      | Privileged | Workload          | `hostNetwork: true` used outside known list                                                                          | Re-architect workload to remove hostNetwork dependency                      |       |             |
|  [ ]   | B-003      | Storage    | Workload          | ReadWriteMany (RWX) PersistentVolumeClaim (PVC) not on Amazon Elastic File System (EFS)                              | Map to Filestore Enterprise or NetApp Volumes                               |       |             |
|  [ ]   | B-004      | Compute    | Cluster           | Karpenter custom NodePools with taints                                                                               | Rebuild as GKE Node Pools or Node Auto-Provisioning (NAP)                   |       |             |
|  [ ]   | B-005      | Network    | Workload          | App Mesh in use                                                                                                      | Re-platform to Managed Service Mesh (Istio)                                 |       |             |
|  [ ]   | B-006      | Identity   | Workload          | Software Development Kit (SDK) calls to Amazon Web Services (AWS) DynamoDB, Kinesis, or Key Management Service (KMS) | Establish cross-cloud VPN or Interconnect, or migrate to Google Cloud       |       |             |
|  [ ]   | B-007      | Registry   | Workload          | Cross-account Amazon Elastic Container Registry (ECR) pulls                                                          | Set up dual-tagging or Artifact Registry pull-through cache                 |       |             |
|  [ ]   | B-008      | SSO        | Platform          | Federated Single Sign-On (SSO) through AWS IAM Identity Center                                                       | Migrate to Google Cloud Workforce Identity Federation                       |       |             |
|  [ ]   | B-009      | Compliance | Platform          | FedRAMP, HIPAA, or PCI compliance scope                                                                              | Verify Google Kubernetes Engine (GKE) service mappings and scope compliance |       |             |

## Resolution history

| Blocker ID | Resolution Details | Resolved By | Date Resolved |
| :--------- | :----------------- | :---------- | :------------ |
|            |                    |             |               |
