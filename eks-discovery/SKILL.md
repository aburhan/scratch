---
name: eks-discovery
description: Inventory an Amazon Elastic Kubernetes Service (EKS) estate end-to-end across compute, workloads, Identity and Access Management (IAM), networking, storage, data dependencies, and observability. Use when starting an EKS-to-Google Kubernetes Engine (GKE) migration, producing a pre-migration inventory, or auditing an EKS environment. Outputs structured JSON and a markdown summary.
metadata:
  produces:
    - inventory.json
    - irsa-map.tsv
    - inventory-summary.md
    - workload-handoff/
---

# EKS Discovery

<!--
  Disabling markdownlint MD029 to provide explicit ordering to avoid
  confusing the LLM.
-->
<!-- markdownlint-disable MD029 -->

You are an Amazon Elastic Kubernetes Service (EKS) inventory specialist. Read
the source Amazon Web Services (AWS) environment with read-only credentials and
produce a complete, machine-readable inventory of everything a migration needs
to know about. You write nothing back to AWS.

## Purpose

Inventory an Amazon EKS estate end-to-end, producing structured outputs for
downstream migration work.

## When to use

- The orchestrator invokes this in Phase 1.
- Starting an EKS-to-Google Kubernetes Engine (GKE) migration that needs a
  baseline inventory.
- User asks "what's in our EKS account" or "audit EKS".
- Refreshing inventory after material changes (new clusters, new workloads, new
  IAM Roles for Service Accounts (IRSA) bindings).

## Prerequisites

- **Read-only AWS credentials** configured through AWS CLI profiles in
  `~/.aws/credentials` or AWS IAM Identity Center (AWS SSO) with permissions
  for: `eks:Describe*`, `eks:List*`, `ec2:Describe*`, `iam:Get*`, `iam:List*`,
  `elasticloadbalancing:Describe*`, `rds:Describe*`, `elasticache:Describe*`,
  `cloudwatch:Describe*`, `logs:Describe*`, `ecr:Describe*`, `ecr:List*`,
  `s3:List*`, `secretsmanager:List*`, `kms:List*`, `kms:Describe*`.
- **Required tools**: `aws` 2.31+, `kubectl` v1.33+, `jq` 1.8+.
- **Required environment variables**:
    - `AWS_REGION` — Target Amazon Web Services (AWS) region to discover
- **Optional environment variables**:
    - `AWS_PROFILE` — AWS CLI profile name (if not using default profile)
    - `CLUSTERS` — Comma-separated list of target EKS clusters (default: all
      discovered)
    - `NAMESPACES` — Comma-separated list of namespaces to include (default: all
      non-system)

### Missing Prerequisites

If any prerequisite is not met, stop immediately. Inform the operator of the
exact missing requirement and instruct them to resolve it.

## CLI conventions

- Every AWS CLI call passes `--region "$AWS_REGION"`.
- Sanitize names containing `/` for filenames: `tr '/' '_'`.
- Write all discovery artifacts under `./01-discovery/`. Per-cluster raw
  artifacts reside in `./01-discovery/raw/<cluster>/` with namespace-scoped
  resources grouped under
  `./01-discovery/raw/<cluster>/namespaces/<namespace>/`.

## Error Handling & Resilience

If an AWS CLI or `kubectl` query fails due to IAM permission limits, network
timeouts, or missing optional APIs, such as `AccessDeniedException` on RDS or
Secrets Manager, do not abort the entire discovery run. Instead:

1.  Record the failure in `escalations.md` (and the `escalations` array of
    `inventory.json`) with a structured escalation entry:

    ```json
    {
      "category": "discovery",
      "code": "permission-denied",
      "message": "Failed to query <service/resource>: <error-details>"
    }
    ```

2.  Proceed immediately with the remaining discovery steps.

## Procedure

### Step 1 — Confirm scope

Verify your active AWS profile and list available clusters in the target region:

```bash
aws sts get-caller-identity
aws eks list-clusters --region "$AWS_REGION"
```

Infer the target regions, accounts, clusters, and namespaces from the user
request or environment variables such as `AWS_REGION`, `CLUSTERS`, and
`NAMESPACES` if available. Default to all discovered clusters and all non-system
namespaces. Skip the confirmation prompts unless there is ambiguity or missing
information.

### Step 2 — Cluster-level facts

For each cluster:

```bash
CLUSTER=...; AWS_REGION=...
RAW_DIR="./01-discovery/raw/$CLUSTER"
mkdir -p "$RAW_DIR"
aws eks update-kubeconfig --region "$AWS_REGION" --name "$CLUSTER"
aws eks describe-cluster --name "$CLUSTER" --region "$AWS_REGION" \
  --query 'cluster.{
    name: name,
    version: version,
    platformVersion: platformVersion,
    endpoint: endpoint,
    publicAccess: resourcesVpcConfig.endpointPublicAccess,
    privateAccess: resourcesVpcConfig.endpointPrivateAccess,
    vpcId: resourcesVpcConfig.vpcId,
    subnetIds: resourcesVpcConfig.subnetIds,
    clusterSGs: resourcesVpcConfig.clusterSecurityGroupId,
    additionalSGs: resourcesVpcConfig.securityGroupIds,
    logging: logging.clusterLogging,
    encryption: encryptionConfig,
    authenticationMode: accessConfig.authenticationMode,
    identity: identity
  }' > "$RAW_DIR/cluster.json"
```

Capture: control-plane version, OIDC provider URL (used for IRSA),
authentication mode (`API`, `API_AND_CONFIG_MAP`, `CONFIG_MAP`), encryption Key
Management Service (KMS) key ARNs, secrets-encryption status.

### Step 3 — Node groups and Karpenter

```bash
aws eks list-nodegroups --cluster-name "$CLUSTER" --region "$AWS_REGION" \
  --query 'nodegroups' --output text | tr '\t' '\n' | while read -r NG; do
    [ -n "$NG" ] && [ "$NG" != "None" ] && \
      aws eks describe-nodegroup --cluster-name "$CLUSTER" --region "$AWS_REGION" \
        --nodegroup-name "$NG" > "$RAW_DIR/ng.$NG.json"
done

aws eks list-fargate-profiles --cluster-name "$CLUSTER" --region "$AWS_REGION" \
  --query 'fargateProfileNames' --output text | tr '\t' '\n' | while read -r FP; do
    [ -n "$FP" ] && [ "$FP" != "None" ] && \
      aws eks describe-fargate-profile --cluster-name "$CLUSTER" \
        --region "$AWS_REGION" --fargate-profile-name "$FP" > "$RAW_DIR/fargate.$FP.json"
done
```

If Karpenter is in use:

```bash
kubectl get nodepool -A -o yaml > "$RAW_DIR/karpenter.nodepool.yaml" 2>/dev/null || true
kubectl get ec2nodeclass -A -o yaml > "$RAW_DIR/karpenter.ec2nodeclass.yaml" 2>/dev/null || true
kubectl get nodeclaims -A > "$RAW_DIR/karpenter.nodeclaims.txt" 2>/dev/null || true
```

Record per node group: instance types, capacity type (`ON_DEMAND`, `SPOT`), AMI
type (`AL2_x86_64`, `BOTTLEROCKET_*`), taints, labels, scaling config, launch
template overrides.

### Step 4 — Addons

```bash
aws eks list-addons --cluster-name "$CLUSTER" --region "$AWS_REGION" \
  --query 'addons' --output text | tr '\t' '\n' | while read -r A; do
    [ -n "$A" ] && [ "$A" != "None" ] && \
      aws eks describe-addon --cluster-name "$CLUSTER" --region "$AWS_REGION" \
        --addon-name "$A" > "$RAW_DIR/addon.$A.json"
done
```

Catalog: `vpc-cni`, `kube-proxy`, `coredns`, `aws-ebs-csi-driver`,
`aws-efs-csi-driver`, `eks-pod-identity-agent`, `cloudwatch-observability`,
`aws-mountpoint-s3-csi-driver`, anything custom.

For each, capture version + configuration overrides (`configurationValues`).

### Step 5 — Workloads (per cluster)

> [!NOTE] Ignore Kubernetes system namespaces (such as `kube-system`,
> `kube-public`, and `kube-node-lease`) by default, unless the user explicitly
> requests to include them.

```bash
# Capture cluster-scoped workloads
for kind in clusterroles clusterrolebindings persistentvolumes storageclasses; do
  kubectl get "$kind" -o yaml > "$RAW_DIR/workloads.$kind.yaml" 2>/dev/null || true
done

kubectl get secrets --all-namespaces -o json 2>/dev/null \
  | jq 'del(.items[].data, .items[].stringData)' > "$RAW_DIR/workloads.secrets-metadata.json" || true

kubectl get crd -o yaml > "$RAW_DIR/workloads.crds.yaml" 2>/dev/null || true

# Capture namespace-scoped workloads grouped by namespace
kubectl get ns -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | while read -r NS; do
  NS_DIR="$RAW_DIR/namespaces/$NS"
  mkdir -p "$NS_DIR"
  for kind in deployments statefulsets daemonsets cronjobs jobs replicasets \
              services ingresses gateways httproutes configmaps serviceaccounts \
              roles rolebindings persistentvolumeclaims networkpolicies \
              poddisruptionbudgets horizontalpodautoscalers verticalpodautoscalers; do
    kubectl get "$kind" -n "$NS" -o yaml > "$NS_DIR/$kind.yaml" 2>/dev/null || true
  done
done
```

Then summarize:

- Total namespaces, with workload counts per namespace.
- Stateful workloads (StatefulSets + Deployments with PVCs).
- DaemonSets (these are platform-level — flag every one).
- Workloads using `hostNetwork`, `hostPID`, `hostPath`, or privileged
  containers.
- Workloads using non-standard `runtimeClassName`.
- Workloads referencing AWS-only fields (annotations beginning with
  `eks.amazonaws.com`, `service.beta.kubernetes.io/aws-`,
  `alb.ingress.kubernetes.io`, `external-dns.alpha.kubernetes.io/aws-`).

### Step 6 — IRSA mapping

For each cluster, list ServiceAccounts annotated with
`eks.amazonaws.com/role-arn`:

```bash
kubectl get sa --all-namespaces -o json \
  | jq -r '.items[] | select(.metadata.annotations["eks.amazonaws.com/role-arn"]) |
    [.metadata.namespace, .metadata.name, .metadata.annotations["eks.amazonaws.com/role-arn"]] | @tsv' \
  > "$RAW_DIR/irsa.tsv" 2>/dev/null || true

aws eks list-pod-identity-associations --cluster-name "$CLUSTER" \
  --region "$AWS_REGION" > "$RAW_DIR/pod-identity-associations.json" 2>/dev/null || true
```

For each role ARN found:

```bash
ROLE=$(echo "$ARN" | awk -F/ '{print $NF}')
aws iam get-role --role-name "$ROLE" > "$RAW_DIR/iam.role.$ROLE.json" 2>/dev/null || true
aws iam list-attached-role-policies --role-name "$ROLE" > "$RAW_DIR/iam.role.$ROLE.attached.json" 2>/dev/null || true
aws iam list-role-policies --role-name "$ROLE" > "$RAW_DIR/iam.role.$ROLE.inline.json" 2>/dev/null || true
for P in $(aws iam list-role-policies --role-name "$ROLE" --query 'PolicyNames' --output text 2>/dev/null); do
  aws iam get-role-policy --role-name "$ROLE" --policy-name "$P" > "$RAW_DIR/iam.role.$ROLE.inline.$P.json" 2>/dev/null || true
done
```

This produces the input to `identity-translation`.

### Step 7 — Networking

```bash
VPC=$(aws eks describe-cluster --name "$CLUSTER" --region "$AWS_REGION" \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text 2>/dev/null)

# Cluster VPC + subnets
[ -n "$VPC" ] && [ "$VPC" != "None" ] && {
  aws ec2 describe-vpcs --vpc-ids "$VPC" > "$RAW_DIR/vpc.$VPC.json" 2>/dev/null || true
  aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC" > "$RAW_DIR/subnets.$VPC.json" 2>/dev/null || true
  aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC" > "$RAW_DIR/routes.$VPC.json" 2>/dev/null || true
  aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC" > "$RAW_DIR/sgs.$VPC.json" 2>/dev/null || true
  aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC" > "$RAW_DIR/nats.$VPC.json" 2>/dev/null || true
  aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC" > "$RAW_DIR/vpcendpoints.$VPC.json" 2>/dev/null || true
  aws ec2 describe-transit-gateway-attachments --filters "Name=resource-id,Values=$VPC" > "$RAW_DIR/tgw.$VPC.json" 2>/dev/null || true
  aws ec2 describe-vpc-peering-connections --filters "Name=requester-vpc-info.vpc-id,Values=$VPC" > "$RAW_DIR/peerings.$VPC.json" 2>/dev/null || true
}

# Load balancers in scope
aws elbv2 describe-load-balancers > "$RAW_DIR/elbv2.json" 2>/dev/null || true
aws elbv2 describe-target-groups > "$RAW_DIR/tgs.json" 2>/dev/null || true
```

Cross-reference: which load balancers are pointed to by which Ingress / Service
in which cluster. The `aws-load-balancer-controller` annotates each provisioned
LB with the originating K8s resource — capture those tags.

### Step 8 — Storage

```bash
kubectl get pv -o yaml > "$RAW_DIR/pv.yaml" 2>/dev/null || true
kubectl get pvc --all-namespaces -o yaml > "$RAW_DIR/pvc.yaml" 2>/dev/null || true
kubectl get storageclass -o yaml > "$RAW_DIR/sc.yaml" 2>/dev/null || true

# EBS volumes attached to the cluster
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/cluster/$CLUSTER,Values=owned" \
  > "$RAW_DIR/ebs.json" 2>/dev/null || true

# EFS file systems referenced
aws efs describe-file-systems > "$RAW_DIR/efs.json" 2>/dev/null || true
aws efs describe-access-points > "$RAW_DIR/efs-aps.json" 2>/dev/null || true
```

For each Persistent Volume (PV), capture: storage class, size, access modes,
reclaim policy, volume handle (Amazon Elastic Block Store (EBS) volume ID or
Amazon Elastic File System (EFS) FS+AP), encryption status, snapshot history.

### Step 9 — Data dependencies

For Amazon Relational Database Service (RDS), Amazon ElastiCache, Amazon Managed
Streaming for Apache Kafka (MSK), Amazon OpenSearch Service, Amazon DynamoDB:

```bash
aws rds describe-db-instances > "$RAW_DIR/rds.json" 2>/dev/null || true
aws rds describe-db-clusters > "$RAW_DIR/rds-clusters.json" 2>/dev/null || true
aws elasticache describe-cache-clusters > "$RAW_DIR/elasticache.json" 2>/dev/null || true
aws elasticache describe-replication-groups > "$RAW_DIR/elasticache-rg.json" 2>/dev/null || true
aws kafka list-clusters-v2 > "$RAW_DIR/msk.json" 2>/dev/null || true
aws opensearch list-domain-names > "$RAW_DIR/opensearch.json" 2>/dev/null || true
aws dynamodb list-tables > "$RAW_DIR/dynamodb.json" 2>/dev/null || true
```

For each, capture: engine + version, instance class, multi-AZ, encryption,
network reachability (subnet group, security groups), backup config, parameter
groups, _which K8s workloads connect to it_ (look up by Secret values, ConfigMap
values, env-from references).

### Step 10 — Registries

```bash
aws ecr describe-repositories > "$RAW_DIR/ecr.json" 2>/dev/null || true
for REPO in $(aws ecr describe-repositories --query 'repositories[].repositoryName' --output text 2>/dev/null); do
  aws ecr describe-images --repository-name "$REPO" \
    --query 'imageDetails[*].{tags:imageTags,digest:imageDigest,pushed:imagePushedAt,size:imageSizeInBytes}' \
    > "$RAW_DIR/ecr.$(echo "$REPO" | tr '/' '_').json" 2>/dev/null || true
done
```

Cross-reference image references in workload manifests so `registry-migration`
knows what's actually used vs cruft.

### Step 11 — Observability

```bash
aws logs describe-log-groups --log-group-name-prefix "/aws/eks/$CLUSTER" > "$RAW_DIR/logs.json" 2>/dev/null || true
aws cloudwatch list-dashboards > "$RAW_DIR/dashboards.json" 2>/dev/null || true
aws cloudwatch describe-alarms --alarm-name-prefix "EKS" > "$RAW_DIR/alarms.json" 2>/dev/null || true
aws aps list-workspaces > "$RAW_DIR/amp.json" 2>/dev/null || true
aws grafana list-workspaces > "$RAW_DIR/amg.json" 2>/dev/null || true
```

Capture: log groups + retention + KMS encryption, dashboards (export JSON),
alarms (export JSON), Amazon Managed Service for Prometheus (AMP) workspaces and
which clusters write to them, Grafana data sources and dashboards.

### Step 12 — Secrets, KMS, certificates

```bash
aws secretsmanager list-secrets > "$RAW_DIR/secrets.json" 2>/dev/null || true
aws kms list-aliases > "$RAW_DIR/kms.json" 2>/dev/null || true
aws acm list-certificates > "$RAW_DIR/acm.json" 2>/dev/null || true
```

Cross-reference: which K8s Secrets are mirrored from Secrets Manager (look for
`secrets-store.csi.x-k8s.io` SecretProviderClass references). Which workloads
reference which KMS key.

## Decision points

- **Sampling vs full enumeration.** For very large clusters (>1k workloads), the
  skill samples by namespace tier (prod / non-prod) and surfaces the sample plan
  to the user. Default: full enumeration.
- **Live mode vs snapshot.** Default: snapshot (no watching, single
  point-in-time). Switch to live only if requested.
- **Cross-account discovery.** If workloads pull from ECR in another account or
  RDS in another account, request read-only access there and discover
  symmetrical. If denied, flag as an escalation.

## Outputs / Deliverables

```text
01-discovery/
├── inventory.json             # Machine-readable input for downstream skills
├── inventory-summary.md       # Human-readable: clusters, workloads, key risks
├── raw/                       # Grouped by cluster and namespace
│   └── <cluster>/
│       ├── cluster.json
│       ├── namespaces/
│       │   └── <namespace>/
│       │       └── <resource>.yaml
│       └── ...
├── workload-handoff/          # Grouped by cluster and namespace
│   └── <cluster>/
│       └── <namespace>/
│           └── workloads.json # Structured handoff matching schema
├── irsa-map.tsv               # IRSA SA → IAM role index
├── data-dependencies.md       # Per-workload upstream data systems
└── escalations.md             # Anything weird (custom CNI, hostNetwork, etc.)
```

See `assets/schemas/` for the authoritative JSON schemas and specifications of
the generated output files.

## Validation

Before declaring discovery complete:

- Every namespace has a workload count, even if zero.
- Every IRSA Service Account (SA) maps to a real, currently-existing IAM role
  (otherwise flag stale binding).
- Every Persistent Volume (PV) resolves to a real EBS volume / EFS access point.
- Every Application Load Balancer (ALB) and Network Load Balancer (NLB) maps
  back to at least one Ingress or Service.
- The number of unique container images in workloads equals the number of
  distinct `(repo, tag-or-digest)` pairs referenced — no orphans, no missing.
- Every escalation in `escalations.md` has a category tag and a one-line
  summary.

## Escalation triggers

Open an escalation (do not silently translate) when discovery finds:

- A custom Container Network Interface (CNI) (Calico in BGP mode, Cilium with
  custom config, anything not VPC CNI). Translation strategy must be
  human-decided.
- A workload using `hostNetwork: true` outside `kube-system`. May not be
  portable as-is.
- A privileged DaemonSet not in a known catalog (cloudwatch-agent, fluent-bit,
  node-problem-detector, csi-node-driver, vpc-cni-node, kube-proxy,
  ebs-csi-node, efs-csi-node).
- An IRSA role with `*` in resources or with cross-account trust relationships.
- A PV with `ReadWriteMany` not on EFS (means a third-party CSI you must catalog
  separately).
- Any workload referencing an AWS-only API directly through an SDK (these will
  need code changes, not config translation).

## Common pitfalls

- **Discovering only what kubectl shows.** Many resources back the cluster —
  IRSA roles, Elastic Load Balancers (ELBs), Amazon Relational Database Service
  (RDS), Amazon Elastic Container Registry (ECR) — and won't appear in kubectl
  get. You must run the AWS CLI portions of each step.
- **Ignoring Custom Resource Definitions (CRDs).** Custom resources (Argo
  Workflows, Crossplane, Karpenter NodeClaims, Istio configurations) are
  first-class workloads. Capture them.
- **Not capturing image digests.** Tags can be re-pushed. Capture digests so the
  registry migration can verify identity.
- **Missing the EKS authentication mode.** If the cluster is `CONFIG_MAP`
  (legacy), the IAM-to-K8s mapping lives in the `aws-auth` ConfigMap and must be
  translated to GKE IAM/RBAC, not just IRSA.
