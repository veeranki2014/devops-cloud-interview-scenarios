# KEDA & Karpenter — Feature Reference Notes

> Quick-reference notes for event-driven pod autoscaling (KEDA) and just-in-time node provisioning (Karpenter) on AWS EKS.

---

## Table of Contents

- [KEDA](#keda)
  - [What it does](#what-keda-does)
  - [Core resources](#keda-core-resources)
  - [AWS authentication methods](#keda-aws-authentication-methods)
  - [Example: SQS-based ScaledObject](#example-sqs-based-scaledobject)
  - [Required IAM permissions](#keda-required-iam-permissions)
- [Karpenter](#karpenter)
  - [What it does](#what-karpenter-does)
  - [Core resources](#karpenter-core-resources)
  - [Dual authentication model](#karpenter-dual-authentication-model)
  - [Example: NodePool + EC2NodeClass](#example-nodepool--ec2nodeclass)
  - [Key features](#karpenter-key-features)
- [KEDA + Karpenter combined flow](#keda--karpenter-combined-flow)
- [EKS Access Entries vs aws-auth ConfigMap](#eks-access-entries-vs-aws-auth-configmap)
- [Real-world use cases](#real-world-use-cases)
- [Useful commands](#useful-commands)
- [References](#references)

---

## KEDA

### What KEDA does

KEDA (Kubernetes Event-Driven Autoscaling) scales the **replica count of pods** based on external event sources — queue depth, stream lag, custom metrics — instead of just CPU/memory.

### KEDA core resources

| Resource | Purpose |
|---|---|
| `ScaledObject` | Defines what to scale and which trigger(s) to use |
| `TriggerAuthentication` | Defines how to authenticate to the external system (namespace-scoped) |
| `ClusterTriggerAuthentication` | Same as above, but cluster-wide |

### KEDA AWS authentication methods

| Method | Use case | Notes |
|---|---|---|
| **IRSA** (`podIdentity.provider: aws-eks`) | EKS, recommended | No static keys; role assumed via OIDC |
| **EKS Pod Identity** | EKS, newer alternative | Simpler setup via `create-pod-identity-association`, no OIDC trust policy needed |
| **Static Secret** (`secretTargetRef`) | Non-EKS clusters (kOps, on-prem) | Access key/secret stored in a K8s Secret; rotate via External Secrets Operator |
| **EC2 Instance Profile** | Fallback | Node-level permissions shared by all pods — least secure, avoid for prod |

`identityOwner` field on the trigger controls **which ServiceAccount's role is used**:
- `operator` — KEDA operator's own ServiceAccount/role (shared across all ScaledObjects)
- `pod` — the target workload's ServiceAccount/role (per-app least privilege)

### Example: SQS-based ScaledObject

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws
spec:
  podIdentity:
    provider: aws-eks
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaledobject
spec:
  scaleTargetRef:
    name: my-worker-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789012/my-queue
        queueLength: "5"
        awsRegion: "us-east-1"
        identityOwner: operator
      authenticationRef:
        name: keda-trigger-auth-aws
```

ServiceAccount used by the operator (IRSA-annotated):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: keda-operator
  namespace: keda
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/keda-sqs-scaler-role
```

### KEDA required IAM permissions

SQS example:

```json
{
  "Effect": "Allow",
  "Action": ["sqs:GetQueueAttributes", "sqs:GetQueueUrl"],
  "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue"
}
```

CloudWatch example: `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics`

---

## Karpenter

### What Karpenter does

Karpenter provisions and terminates **EC2 nodes** just-in-time based on actual pending pod requirements — no pre-defined node groups or ASGs. It replaces (or complements) Cluster Autoscaler.

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Scales | Existing ASGs / node groups | Direct EC2 API calls |
| Instance selection | Fixed per node group | Right-sized per pending pod batch |
| Speed | Slower (ASG scaling activity) | Faster (direct `RunInstances`/`CreateFleet`) |
| Bin-packing | Limited | Built-in, cost-aware |
| Consolidation | No | Yes — auto-repacks/terminates underutilized nodes |

### Karpenter core resources

| Resource | Purpose |
|---|---|
| `NodePool` | Constraints: instance families, arch, capacity type (spot/on-demand), taints, limits, disruption policy |
| `EC2NodeClass` (AWS) | Infra details: AMI family, subnets, security groups, instance profile |

### Karpenter dual authentication model

Karpenter needs **two separate IAM identities** — don't confuse them:

1. **Controller identity (IRSA)** — the Karpenter controller pod calls EC2 APIs (`RunInstances`, `CreateFleet`, `TerminateInstances`, `DescribeInstanceTypes`, pricing API, SSM for AMI resolution).
2. **Node identity (instance profile)** — each *launched EC2 instance* needs a node IAM role (attached via instance profile) so kubelet can authenticate to the EKS control plane. This is mapped to Kubernetes RBAC via **EKS Access Entries** (or the legacy `aws-auth` ConfigMap).

```
Controller path:
  Karpenter pod (IRSA) → Controller IAM role → EC2 RunInstances → new node

Node bootstrap path:
  New EC2 node → Instance profile → Node IAM role
     → EKS Access Entry (maps role to system:nodes)
     → kubelet joins cluster → node Ready
```

Controller IAM policy (trimmed):

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:RunInstances",
    "ec2:CreateFleet",
    "ec2:TerminateInstances",
    "ec2:DescribeInstances",
    "ec2:DescribeInstanceTypes",
    "ec2:DescribeLaunchTemplates",
    "pricing:GetProducts",
    "ssm:GetParameter"
  ],
  "Resource": "*"
}
```

ServiceAccount for the controller:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: karpenter
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/KarpenterControllerRole-my-cluster
```

> ⚠️ If you annotate a ServiceAccount *after* a pod is already running, the webhook won't retroactively inject the token — restart the pod.

### Example: NodePool + EC2NodeClass

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  role: KarpenterNodeRole-my-cluster
```

### Karpenter key features

- **Bin-packing at launch** — picks the cheapest instance type from a candidate list that fits all pending pods, mixing instance families in one pool
- **Spot-to-on-demand fallback** — tries spot first, falls back automatically using AWS Spot placement scores
- **Consolidation** — proactively terminates/replaces underutilized nodes by repacking pods onto fewer/cheaper nodes
- **Drift detection** — flags and replaces nodes when NodePool/EC2NodeClass config changes (e.g., new AMI)
- **Fast scale-up** — calls EC2 `RunInstances`/`CreateFleet` directly, skipping ASG scaling-activity latency

---

## KEDA + Karpenter combined flow

| Layer | Tool | Trigger |
|---|---|---|
| Pods (replica count) | KEDA | External event source (SQS depth, Kafka lag, CloudWatch metric) |
| Nodes (compute capacity) | Karpenter | Unschedulable pods needing resources |

**Typical sequence:**
1. SQS queue backs up
2. KEDA polls queue (via IRSA), scales Deployment to N replicas
3. New pods go `Pending` — insufficient node capacity
4. Karpenter sees pending pods, calls `CreateFleet` for right-sized nodes
5. New nodes join cluster (node IAM role + access entry), pods scheduled
6. Queue drains → KEDA scales pods back down
7. Karpenter consolidates/terminates now-idle nodes

---

## EKS Access Entries vs `aws-auth` ConfigMap

Both map an IAM identity (role/user) to a Kubernetes RBAC identity so it can authenticate to the EKS API server. Access Entries are the modern replacement for the legacy `aws-auth` ConfigMap.

### `aws-auth` ConfigMap (legacy)

A hand-edited Kubernetes ConfigMap in `kube-system` mapping IAM ARNs to K8s usernames/groups.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/KarpenterNodeRole-my-cluster
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/DevOpsAdminRole
      username: devops-admin
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/jane
      username: jane
      groups:
        - system:masters
```

**Auth flow:** `kubectl`/kubelet signs request via SigV4 → EKS webhook calls STS `GetCallerIdentity` → looks up ARN in `aws-auth` → maps to K8s user/group → normal RBAC applies.

**Problems:**
- Plain YAML, no schema validation — typos fail silently
- No CloudTrail audit trail (only K8s audit logs, if enabled)
- Lockout risk — a bad edit with no other admin path in can lock you out of the cluster entirely
- Concurrent edits from multiple tools (Terraform, eksctl, manual) cause race conditions
- No native rollback beyond your own version control

### EKS Access Entries (modern, GA ~2023)

A first-class EKS API resource — managed via AWS CLI/Console/Terraform, independent of `kubectl` access.

```bash
# Node role
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/KarpenterNodeRole-my-cluster \
  --type EC2_LINUX

# Human admin
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsAdminRole \
  --type STANDARD

# Attach an AWS-managed permission policy
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevOpsAdminRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

Terraform:

```hcl
resource "aws_eks_access_entry" "karpenter_node" {
  cluster_name  = "my-cluster"
  principal_arn = aws_iam_role.karpenter_node.arn
  type          = "EC2_LINUX"
}

resource "aws_eks_access_entry" "devops_admin" {
  cluster_name  = "my-cluster"
  principal_arn = aws_iam_role.devops_admin.arn
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "devops_admin_policy" {
  cluster_name  = "my-cluster"
  principal_arn = aws_iam_role.devops_admin.arn
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  access_scope {
    type = "cluster"
  }
}
```

**Access entry types:**

| Type | Purpose |
|---|---|
| `STANDARD` | Human user or generic IAM role |
| `EC2_LINUX` / `EC2_WINDOWS` | Self-managed node group / Karpenter node role |
| `FARGATE_LINUX` | Fargate pod execution role |

**AWS-managed access policies** (replace hand-written `ClusterRoleBindings`):

- `AmazonEKSClusterAdminPolicy` — full cluster admin
- `AmazonEKSAdminPolicy` — admin minus some cluster-level settings
- `AmazonEKSEditPolicy` — edit most resources, no RBAC changes
- `AmazonEKSViewPolicy` — read-only

You can still layer native K8s RBAC on top for finer granularity — access entries replace the *authentication mapping*, not RBAC itself.

### Comparison table

| Aspect | `aws-auth` ConfigMap | EKS Access Entries |
|---|---|---|
| Storage | Kubernetes ConfigMap (YAML) | Native EKS API object |
| Managed via | `kubectl edit/apply` | AWS CLI, Console, Terraform/CloudFormation |
| Validation | None — silent typo failures | API-validated ARNs and types |
| Audit trail | K8s audit logs only (if enabled) | Full CloudTrail record |
| Lockout risk | High | Low — manageable even if `kubectl` access is broken |
| Requires existing `kubectl` access to edit | Yes | No — just `eks:CreateAccessEntry` IAM permission |
| RBAC binding | Manual `groups:` + `ClusterRoleBindings` | AWS-managed policies, or custom RBAC on top |
| IaC-friendly | Awkward (YAML-in-YAML) | Native Terraform/CloudFormation resources |

### Cluster authentication modes

| Mode | Behavior |
|---|---|
| `CONFIG_MAP` | Legacy-only — only `aws-auth` honored |
| `API_AND_CONFIG_MAP` | Migration mode — both checked, access entries win on conflict |
| `API` | Modern-only — `aws-auth` ignored entirely |

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP
```

### Migration path

1. Switch cluster to `API_AND_CONFIG_MAP` (non-disruptive, both coexist)
2. Recreate every `aws-auth` entry as an equivalent access entry (nodes, CI/CD roles, human admins)
3. Verify access for every consumer — test each role individually
4. Switch to `API` mode once confident
5. Delete the `aws-auth` ConfigMap (optional cleanup)

> ⚠️ **Gotcha**: nodes created before a cluster supported access entries were auto-mapped into `aws-auth` by EKS. When migrating, explicitly create an `EC2_LINUX` access entry for Karpenter's node role — don't assume it carries over silently.

### Recommendation

- **New clusters**: start directly in `API` mode, skip `aws-auth` entirely
- **Existing clusters**: migrate opportunistically via `API_AND_CONFIG_MAP`
- **Karpenter**: v0.32+/v1 docs and Helm defaults assume access entries — migrate older setups when convenient

---

## Real-world use cases

- **E-commerce flash sale** — SQS backlog spikes 20x → KEDA scales checkout workers → Karpenter launches spot `c5.4xlarge` nodes in seconds → consolidates back down once the sale traffic tapers
- **ML/batch training** — Kubeflow/Argo job requests GPU nodes only when submitted; Karpenter provisions `p3.2xlarge` on demand, terminates immediately after job completion
- **Multi-tenant SaaS** — per-tenant NodePools (dedicated on-demand + taints for enterprise, shared spot for free tier)
- **CI/CD runners** — self-hosted GitHub Actions runners scale nodes only while jobs are queued, terminate right after

---

## Useful commands

```bash
# Associate OIDC provider with an EKS cluster (required for IRSA)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Check current Karpenter NodePools
kubectl get nodepools

# Check current Karpenter nodeclaims (in-flight provisioning)
kubectl get nodeclaims

# Describe a KEDA ScaledObject and current scaling status
kubectl describe scaledobject sqs-scaledobject

# Check KEDA operator logs for auth issues
kubectl logs -n keda deployment/keda-operator

# Check Karpenter controller logs
kubectl logs -n kube-system deployment/karpenter
```

---

## References

- [KEDA docs](https://keda.sh/docs/)
- [KEDA AWS SQS scaler](https://keda.sh/docs/latest/scalers/aws-sqs/)
- [Karpenter docs](https://karpenter.sh/docs/)
- [AWS IRSA docs](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)