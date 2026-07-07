---
name: eks-discovery
description: Inventory an Amazon Elastic Kubernetes Service (EKS) estate end-to-end across compute, workloads, Identity and Access Management (IAM), networking, Domain Name System (DNS), storage, data dependencies, and observability. Use when starting a migration from Amazon EKS to Google Kubernetes Engine (GKE), producing a pre-migration inventory, or auditing an EKS environment. Outputs structured JSON and a markdown summary.
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

Inventory an Amazon EKS estate end-to-end and produce structured outputs for
downstream migration work.

## When to use

- The orchestrator invokes this in Phase 1.
- You start an EKS-to-GKE migration that needs a baseline inventory.
- The user asks "what's in our EKS account" or "audit EKS".
- You refresh the inventory after material changes such as new clusters, new
  workloads, or new Identity and Access Management (IAM) Roles for Service
  Accounts (SAs) bindings. This document refers to IAM Roles for Service
  Accounts as IRSA.

## Prerequisites

- **Read-only Amazon Web Services (AWS) credentials** configured through AWS CLI
  profiles in `~/.aws/credentials` or AWS IAM Identity Center, with permissions
  for: `eks:Describe*`, `eks:List*`, `ec2:Describe*`, `iam:Get*`, `iam:List*`,
  `elasticloadbalancing:Describe*`, `route53:List*`, `route53:Get*`,
  `rds:Describe*`, `elasticache:Describe*`, `cloudwatch:Describe*`,
  `logs:Describe*`, `ecr:Describe*`, `ecr:List*`, `s3:List*`,
  `secretsmanager:List*`, `kms:List*`, `kms:Describe*`.
- **Required tools**: `aws` 2.31+, `kubectl` v1.33+, `jq` 1.8+.
- **Required environment variables**:
    - `AWS_REGION` — Target Amazon Web Services region to discover.
- **Optional environment variables**:
    - `AWS_PROFILE` — AWS CLI profile name when you do not use the default
      profile.
    - `CLUSTERS` — Comma-separated list of target EKS clusters. The default is
      all discovered clusters.
    - `NAMESPACES` — Comma-separated list of namespaces to include in the
      synthesis outputs of Step 13. The default is all non-system namespaces.
      This filter never applies to raw capture. See Step 5.

### Missing prerequisites

If you cannot meet any prerequisite, stop immediately. Tell the operator the
exact missing requirement and instruct them to resolve it.

## CLI conventions

- Pass `--region "$AWS_REGION"` on every AWS CLI call.
- Sanitize names containing `/` for filenames with `tr '/' '_'`.
- Write all discovery artifacts under `./01-discovery/`. Per-cluster raw
  artifacts reside in `./01-discovery/raw/<cluster>/`. Group namespace-scoped
  resources under `./01-discovery/raw/<cluster>/namespaces/<namespace>/`.

## Error handling and resilience

When an AWS CLI or `kubectl` query fails due to IAM permission limits, network
timeouts, or missing optional APIs, such as `AccessDeniedException` on RDS or
Secrets Manager, do not abort the entire discovery run. Instead:

1.  Record the failure in `escalations.md` and in the `escalations` array of
    `inventory.json` with a structured escalation entry:

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
request or from environment variables such as `AWS_REGION`, `CLUSTERS`, and
`NAMESPACES` when available. Default to all discovered clusters and all
non-system namespaces. Skip the confirmation prompts unless you find ambiguity
or missing information.

### Step 2 — Cluster-level facts

For each cluster:

```bash
CLUSTER=...
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

Capture the control-plane version, the OpenID Connect (OIDC) provider URL that
IRSA uses, the authentication mode — `API`, `API_AND_CONFIG_MAP`, or
`CONFIG_MAP` — the Key Management Service (KMS) key Amazon Resource Names
(ARNs), and the secrets-encryption status.

### Step 3 — Node groups and Karpenter

```bash
aws eks list-nodegroups --cluster-name "$CLUSTER" \
  --region "$AWS_REGION" --query 'nodegroups' --output text | \
  tr '\t' '\n' | while read -r NG
  do
    [ -n "$NG" ] && [ "$NG" != "None" ] && \
      aws eks describe-nodegroup --cluster-name "$CLUSTER" \
        --region "$AWS_REGION" --nodegroup-name "$NG" \
        > "$RAW_DIR/ng.$NG.json"
done

aws eks list-fargate-profiles --cluster-name "$CLUSTER" \
  --region "$AWS_REGION" --query 'fargateProfileNames' --output text | \
  tr '\t' '\n' | while read -r FP
  do
    [ -n "$FP" ] && [ "$FP" != "None" ] && \
      aws eks describe-fargate-profile --cluster-name "$CLUSTER" \
        --region "$AWS_REGION" --fargate-profile-name "$FP" \
        > "$RAW_DIR/fargate.$FP.json"
done
```

When Karpenter is in use:

```bash
kubectl get nodepool -A -o yaml \
  > "$RAW_DIR/karpenter.nodepool.yaml" 2>/dev/null || true
kubectl get ec2nodeclass -A -o yaml \
  > "$RAW_DIR/karpenter.ec2nodeclass.yaml" 2>/dev/null || true
kubectl get nodeclaims -A \
  > "$RAW_DIR/karpenter.nodeclaims.txt" 2>/dev/null || true
```

Record per node group: instance types, capacity type such as `ON_DEMAND` or
`SPOT`, Amazon Machine Image (AMI) type such as `AL2_x86_64` or
`BOTTLEROCKET_*`, taints, labels, scaling config, and launch template overrides.

### Step 4 — Addons

```bash
aws eks list-addons --cluster-name "$CLUSTER" --region "$AWS_REGION" \
  --query 'addons' --output text | tr '\t' '\n' | while read -r A
  do
    [ -n "$A" ] && [ "$A" != "None" ] && \
      aws eks describe-addon --cluster-name "$CLUSTER" --region "$AWS_REGION" \
        --addon-name "$A" > "$RAW_DIR/addon.$A.json"
done
```

Catalog `vpc-cni`, `kube-proxy`, `coredns`, `aws-ebs-csi-driver`,
`aws-efs-csi-driver`, `eks-pod-identity-agent`, `cloudwatch-observability`,
`aws-mountpoint-s3-csi-driver`, and anything custom.

For each, capture the version and the configuration overrides in
`configurationValues`.

### Step 5 — Workloads, per cluster

> [!NOTE] Raw capture includes **every** namespace, system namespaces included.
> The loop below is intentionally unfiltered. `kube-system` holds data this
> skill's own outputs depend on: the `aws-auth` ConfigMap, IRSA bindings for the
> load-balancer controller and CSI drivers, and the DaemonSet catalog the
> escalation triggers evaluate. Apply the `NAMESPACES` filter — which excludes
> system namespaces by default — only when you build the synthesis outputs in
> Step 13, and record the applied filter in the `source_aws.namespace_filter`
> field of `inventory.json`.

```bash
# Capture cluster-scoped workloads
for kind in clusterroles clusterrolebindings persistentvolumes storageclasses
do
  kubectl get "$kind" -o yaml \
    > "$RAW_DIR/workloads.$kind.yaml" 2>/dev/null || true
done

kubectl get secrets --all-namespaces -o json 2>/dev/null | jq \
  'del(.items[].data, .items[].stringData)' \
  > "$RAW_DIR/workloads.secrets-metadata.json" || true

kubectl get crd -o yaml > "$RAW_DIR/workloads.crds.yaml" 2>/dev/null || true

# Capture custom resource INSTANCES. Definitions alone do not satisfy the
# "Ignoring Custom Resource Definitions (CRDs)" pitfall. Argo, Crossplane,
# Karpenter, ENIConfig, and
# SecretProviderClass instances are first-class migration inputs.
kubectl api-resources --verbs=list --namespaced=false -o name 2>/dev/null \
  | grep '\.' | while read -r R
  do
    kubectl get "$R" -o yaml \
      >> "$RAW_DIR/workloads.cr-instances.cluster.yaml" 2>/dev/null || true
done
kubectl api-resources --verbs=list --namespaced=true -o name 2>/dev/null \
  | grep '\.' | while read -r R
  do
    kubectl get "$R" --all-namespaces -o yaml \
      >> "$RAW_DIR/workloads.cr-instances.namespaced.yaml" 2>/dev/null || true
done

# Capture the aws-auth ConfigMap. The legacy IAM-to-Kubernetes mapping
# lives here and identity-translation consumes it.
kubectl get cm aws-auth -n kube-system -o yaml \
  > "$RAW_DIR/aws-auth.yaml" 2>/dev/null || true

# Capture namespace-scoped workloads grouped by namespace
kubectl get ns -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' \
  | tee "$RAW_DIR/namespaces.enumerated.txt" | while read -r NS
  do
  NS_DIR="$RAW_DIR/namespaces/$NS"
  mkdir -p "$NS_DIR"
  for kind in deployments statefulsets daemonsets cronjobs jobs replicasets \
              services ingresses gateways httproutes configmaps \
              serviceaccounts roles rolebindings persistentvolumeclaims \
              networkpolicies poddisruptionbudgets \
              horizontalpodautoscalers verticalpodautoscalers
  do
    kubectl get "$kind" -n "$NS" -o yaml \
      > "$NS_DIR/$kind.yaml" 2>/dev/null || true
  done
done
```

Then summarize:

- Total namespaces, with workload counts per namespace.
- Stateful workloads, meaning StatefulSets plus Deployments with PVCs.
- DaemonSets. These are platform-level — flag every one.
- Workloads using `hostNetwork`, `hostPID`, `hostPath`, or privileged
  containers.
- Workloads using non-standard `runtimeClassName`.
- Workloads referencing AWS-only fields, meaning annotations beginning with
  `eks.amazonaws.com`, `service.beta.kubernetes.io/aws-`,
  `alb.ingress.kubernetes.io`, or `external-dns.alpha.kubernetes.io/aws-`.

### Step 6 — IRSA mapping

For each cluster, list ServiceAccounts annotated with
`eks.amazonaws.com/role-arn`:

```bash
kubectl get sa --all-namespaces -o json | jq -r \
  '.items[] | select(.metadata.annotations["eks.amazonaws.com/role-arn"]) |
   [.metadata.namespace, .metadata.name,
    .metadata.annotations["eks.amazonaws.com/role-arn"]] | @tsv' \
  > "$RAW_DIR/irsa.tsv" 2>/dev/null || true

aws eks list-pod-identity-associations --cluster-name "$CLUSTER" \
  --region "$AWS_REGION" \
  > "$RAW_DIR/pod-identity-associations.json" 2>/dev/null || true
```

For each **unique** role ARN in `irsa.tsv`, dedupe first — multiple
ServiceAccounts can share one role:

```bash
cut -f3 "$RAW_DIR/irsa.tsv" | sort -u | while read -r ARN
do
  ROLE=$(echo "$ARN" | awk -F/ '{print $NF}')
  aws iam get-role --role-name "$ROLE" \
    > "$RAW_DIR/iam.role.$ROLE.json" 2>/dev/null || true
  aws iam list-attached-role-policies --role-name "$ROLE" \
    > "$RAW_DIR/iam.role.$ROLE.attached.json" 2>/dev/null || true
  aws iam list-role-policies --role-name "$ROLE" \
    > "$RAW_DIR/iam.role.$ROLE.inline.json" 2>/dev/null || true
  for P in $(aws iam list-role-policies --role-name "$ROLE" \
    --query 'PolicyNames' --output text 2>/dev/null)
  do
    aws iam get-role-policy --role-name "$ROLE" --policy-name "$P" \
      > "$RAW_DIR/iam.role.$ROLE.inline.$P.json" 2>/dev/null || true
  done
done
```

This produces the input to `identity-translation`.

### Step 7 — Networking

```bash
VPC=$(aws eks describe-cluster --name "$CLUSTER" --region "$AWS_REGION" \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text 2>/dev/null)

# Cluster Virtual Private Cloud (VPC) + subnets
[ -n "$VPC" ] && [ "$VPC" != "None" ] && {
  aws ec2 describe-vpcs --vpc-ids "$VPC" \
    > "$RAW_DIR/vpc.$VPC.json" 2>/dev/null || true
  aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC" \
    > "$RAW_DIR/subnets.$VPC.json" 2>/dev/null || true
  aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC" \
    > "$RAW_DIR/routes.$VPC.json" 2>/dev/null || true
  aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC" \
    > "$RAW_DIR/sgs.$VPC.json" 2>/dev/null || true
  aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC" \
    > "$RAW_DIR/nats.$VPC.json" 2>/dev/null || true
  aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC" \
    > "$RAW_DIR/vpcendpoints.$VPC.json" 2>/dev/null || true
  aws ec2 describe-transit-gateway-attachments \
    --filters "Name=resource-id,Values=$VPC" \
    > "$RAW_DIR/tgw.$VPC.json" 2>/dev/null || true
  aws ec2 describe-vpc-peering-connections \
    --filters "Name=requester-vpc-info.vpc-id,Values=$VPC" \
    > "$RAW_DIR/peerings.$VPC.json" 2>/dev/null || true
}

# Load balancers in scope
aws elbv2 describe-load-balancers > "$RAW_DIR/elbv2.json" 2>/dev/null || true
aws elbv2 describe-target-groups > "$RAW_DIR/tgs.json" 2>/dev/null || true

# Amazon Route 53 — source data for network-translation's Domain Name System
# (DNS) plan
aws route53 list-hosted-zones \
  > "$RAW_DIR/route53-zones.json" 2>/dev/null || true
jq -r '.HostedZones[].Id' "$RAW_DIR/route53-zones.json" 2>/dev/null | \
  while read -r Z
do
  SAFE=$(echo "$Z" | tr '/' '_')
  aws route53 list-resource-record-sets --hosted-zone-id "$Z" \
    > "$RAW_DIR/route53-records.$SAFE.json" 2>/dev/null || true
done
```

Cross-reference which load balancers each Ingress or Service in each cluster
points to. The `aws-load-balancer-controller` annotates each provisioned load
balancer with the originating Kubernetes resource — capture those tags. Also
cross-reference which Route 53 records point at discovered load balancer DNS
names through alias targets and CNAMEs. These are the records
`network-translation` must plan Cloud DNS equivalents for.

### Step 8 — Storage

```bash
kubectl get pv -o yaml > "$RAW_DIR/pv.yaml" 2>/dev/null || true
kubectl get pvc --all-namespaces -o yaml \
  > "$RAW_DIR/pvc.yaml" 2>/dev/null || true
kubectl get storageclass -o yaml > "$RAW_DIR/sc.yaml" 2>/dev/null || true

# EBS volumes attached to the cluster
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/cluster/$CLUSTER,Values=owned" \
  > "$RAW_DIR/ebs.json" 2>/dev/null || true

# EFS file systems referenced
aws efs describe-file-systems > "$RAW_DIR/efs.json" 2>/dev/null || true
aws efs describe-access-points > "$RAW_DIR/efs-aps.json" 2>/dev/null || true
```

For each Persistent Volume (PV), capture the storage class, size, access modes,
reclaim policy, volume handle — the Amazon Elastic Block Store (EBS) volume ID
or the Amazon Elastic File System (EFS) filesystem and access point — encryption
status, and snapshot history.

### Step 9 — Data dependencies

Query Amazon Relational Database Service (RDS), Amazon ElastiCache, Amazon
Managed Streaming for Apache Kafka (MSK), Amazon OpenSearch Service, and Amazon
DynamoDB:

```bash
aws rds describe-db-instances > "$RAW_DIR/rds.json" 2>/dev/null || true
aws rds describe-db-clusters > "$RAW_DIR/rds-clusters.json" 2>/dev/null || true
aws elasticache describe-cache-clusters \
  > "$RAW_DIR/elasticache.json" 2>/dev/null || true
aws elasticache describe-replication-groups \
  > "$RAW_DIR/elasticache-rg.json" 2>/dev/null || true
aws kafka list-clusters-v2 > "$RAW_DIR/msk.json" 2>/dev/null || true
aws opensearch list-domain-names \
  > "$RAW_DIR/opensearch.json" 2>/dev/null || true
aws dynamodb list-tables > "$RAW_DIR/dynamodb.json" 2>/dev/null || true
```

For each, capture the engine and version, instance class, multi-AZ status,
encryption, network reachability including the subnet group and security groups,
backup config, parameter groups, and _which Kubernetes workloads connect to it_.
Resolve the workload connections through Secret values, ConfigMap values, and
env-from references.

### Step 10 — Registries

```bash
aws ecr describe-repositories > "$RAW_DIR/ecr.json" 2>/dev/null || true
for REPO in $(aws ecr describe-repositories \
  --query 'repositories[].repositoryName' --output text 2>/dev/null)
do
  aws ecr describe-images --repository-name "$REPO" \
    --query 'imageDetails[*].{
      tags: imageTags,
      digest: imageDigest,
      pushed: imagePushedAt,
      size: imageSizeInBytes
    }' > "$RAW_DIR/ecr.$(echo "$REPO" | tr '/' '_').json" 2>/dev/null || true
done
```

Cross-reference image references in workload manifests so `registry-migration`
knows what's actually used vs cruft.

### Step 11 — Observability

```bash
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/eks/$CLUSTER" \
  > "$RAW_DIR/logs.json" 2>/dev/null || true
aws cloudwatch list-dashboards > "$RAW_DIR/dashboards.json" 2>/dev/null || true
aws cloudwatch describe-alarms --alarm-name-prefix "EKS" \
  > "$RAW_DIR/alarms.json" 2>/dev/null || true
aws aps list-workspaces > "$RAW_DIR/amp.json" 2>/dev/null || true
aws grafana list-workspaces > "$RAW_DIR/amg.json" 2>/dev/null || true
```

Capture the log groups with retention and KMS encryption, the dashboards as
exported JSON, the alarms as exported JSON, the Amazon Managed Service for
Prometheus workspaces and which clusters write to them, and the Grafana data
sources and dashboards.

### Step 12 — Secrets, KMS, certificates

```bash
aws secretsmanager list-secrets > "$RAW_DIR/secrets.json" 2>/dev/null || true
aws kms list-aliases > "$RAW_DIR/kms.json" 2>/dev/null || true
aws acm list-certificates > "$RAW_DIR/acm.json" 2>/dev/null || true
```

Cross-reference which Kubernetes Secrets mirror Secrets Manager entries — look
for `secrets-store.csi.x-k8s.io` SecretProviderClass references — and which
workloads reference which KMS key.

### Step 13 — Synthesize outputs

Build the derived artifacts by hand from the raw files, conforming exactly to
`assets/schemas/`:

- `inventory.json` per `inventory.schema.json`. Every field maps to a raw file.
  Do not invent values. Apply the `NAMESPACES` filter here and record it in
  `source_aws.namespace_filter`.
- `workload-handoff/<cluster>/<namespace>/workloads.json` per
  `workload-handoff-namespace.schema.json`, one file per in-scope namespace.
- `irsa-map.tsv` per `irsa-map.schema.md`: every `irsa.tsv` row plus every Pod
  Identity association. Run `describe-pod-identity-association` per association
  from Step 6's list output — the list call omits the role ARN.
- `inventory-summary.md` and `data-dependencies.md`.

### Step 14 — Validate

Run every check in the Validation section below and record the results in
`01-discovery/validation-report.md`: one row per check with the exact command
you ran, its observed output, and `PASS` or `FAIL`. Discovery is not complete
until every row shows `PASS` or shows `FAIL` with a linked escalation. Add this
coverage check — it catches silent capture gaps. The output must be empty for
every cluster:

```bash
diff <(sort "$RAW_DIR/namespaces.enumerated.txt") \
  <(ls "$RAW_DIR/namespaces" | sort)
```

## Decision points

- **Sampling vs full enumeration.** For very large clusters with over 1k
  workloads, the skill samples by namespace tier — prod and non-prod — and
  surfaces the sample plan to the user. Default: full enumeration.
- **Live mode vs snapshot.** Default: snapshot, a single point-in-time with no
  watching. Switch to live only when requested.
- **Cross-account discovery.** When workloads pull from Amazon Elastic Container
  Registry (ECR) in another account or RDS in another account, request read-only
  access there and discover symmetrically. If denied, flag as an escalation.

## Outputs / deliverables

```text
01-discovery/
├── inventory.json             # Machine-readable input for downstream skills
├── inventory-summary.md       # Human-readable: clusters, workloads, key risks
├── validation-report.md       # Step 14 evidence. Zero FAILs = complete
├── raw/                       # Grouped by cluster and namespace
│   └── <cluster>/
│       ├── cluster.json
│       ├── namespaces.enumerated.txt
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
└── escalations.md             # Anything weird: custom CNI, hostNetwork, etc.
```

See `assets/schemas/` for the authoritative JSON schemas and specifications of
the generated output files.

## Validation

Before declaring discovery complete:

- Every namespace has a workload count, even if zero.
- Every IRSA ServiceAccount maps to a real, currently-existing IAM role.
  Otherwise flag a stale binding.
- Every Persistent Volume resolves to a real EBS volume or EFS access point.
- Every Application Load Balancer (ALB) and Network Load Balancer (NLB) maps
  back to at least one Ingress or Service.
- The number of unique container images in workloads equals the number of
  distinct repo and tag-or-digest pairs referenced — no orphans, no missing.
- Every escalation in `escalations.md` has a category tag and a one-line
  summary.

## Escalation triggers

Open an escalation — do not silently translate — when discovery finds:

- A custom Container Network Interface, such as Calico in BGP mode, Cilium with
  custom config, or anything not VPC CNI. A human must decide the translation
  strategy.
- A workload using `hostNetwork: true` outside `kube-system`. It may not port
  as-is.
- A privileged DaemonSet not in a known catalog: cloudwatch-agent, fluent-bit,
  node-problem-detector, csi-node-driver, vpc-cni-node, kube-proxy,
  ebs-csi-node, efs-csi-node.
- An IRSA role with `*` in resources or with cross-account trust relationships.
- A PV with `ReadWriteMany` not on EFS. This means a third-party CSI you must
  catalog separately.
- Any workload calling an AWS-only API directly through an SDK. These need code
  changes, not config translation.

## Common pitfalls

- **Discovering only what kubectl shows.** Many resources back the cluster —
  IRSA roles, ALBs, NLBs, RDS, ECR — and never appear in kubectl get. You must
  run the AWS CLI portions of each step.
- **Filtering system namespaces out of raw capture.** The `aws-auth` ConfigMap,
  platform IRSA bindings, and the DaemonSet catalog live in `kube-system`.
  Filter at synthesis in Step 13, never at capture.
- **Ignoring CRDs.** Custom resources — Argo Workflows, Crossplane, Karpenter
  NodeClaims, Istio configurations — are first-class workloads. Capture both the
  definitions and the instances. Step 5 does both.
- **Not capturing image digests.** Anyone can re-push a tag. Capture digests so
  the registry migration can verify identity.
- **Missing the EKS authentication mode.** When the cluster is `CONFIG_MAP`, the
  legacy mode, the IAM-to-Kubernetes mapping lives in the `aws-auth` ConfigMap
  and you must translate it to GKE IAM and RBAC, not just IRSA.
