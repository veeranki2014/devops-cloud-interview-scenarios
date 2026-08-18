# Senior DevOps / Platform Engineer Interview Guide (16+ Years Experience)

Target JD areas: Terraform, AWS & Networking, Ansible, GitOps & CI/CD

At this experience level, interviewers expect **architectural reasoning, trade-off analysis, and war stories** — not textbook definitions. Each answer below is framed the way a 16+ year candidate should respond: concise conceptual anchor, then depth, then a real-world nuance or gotcha.

---

## Section 1: Terraform

### Q1. How do you design a custom Terraform module for reuse across multiple teams/environments?
**Answer:**
A good custom module treats itself like a mini API contract:
- **Inputs (variables.tf):** only expose what genuinely needs to vary — instance sizes, environment, tags, feature flags. Avoid exposing every provider attribute; that turns the module into a pass-through and defeats the purpose of abstraction.
- **Outputs (outputs.tf):** expose IDs/ARNs/endpoints that downstream modules or root configs will consume (e.g., VPC ID, subnet IDs, security group IDs).
- **Sensible defaults:** use `variable "x" { default = ... }` so 80% of consumers don't need to override anything.
- **Validation blocks:** use `validation {}` on variables to fail fast on bad input (e.g., CIDR format, allowed instance types).
- **Versioning:** publish modules to a private registry (Terraform Cloud/Enterprise, or a Git tag-based source `?ref=v1.4.0`) so consumers pin versions and upgrades are opt-in, not forced.
- **Composition over monoliths:** keep modules single-purpose (one for VPC, one for EKS, one for RDS) and compose them at the root/environment level rather than building one giant "do everything" module.

I also enforce `terraform-docs` to auto-generate README documentation from variables/outputs so module consumers don't have to read source code.

---

### Q2. Explain a practical use case of `for_each` with a map of objects. Why prefer it over `count`?
**Answer:**
`count` indexes resources by number (0,1,2...), so if you remove an item from the middle of a list, Terraform sees a diff in every subsequent index and wants to destroy/recreate resources unnecessarily. `for_each` indexes by **key**, so removing one entry only affects that one resource — much safer for stateful infra like EC2 instances, IAM users, or S3 buckets.

Example — creating multiple IAM roles with different policies from a map of objects:

```hcl
variable "iam_roles" {
  type = map(object({
    description = string
    policy_arn  = string
  }))
}

resource "aws_iam_role" "this" {
  for_each = var.iam_roles

  name        = each.key
  description = each.value.description
  assume_role_policy = data.aws_iam_policy_document.trust.json
}

resource "aws_iam_role_policy_attachment" "this" {
  for_each   = var.iam_roles
  role       = aws_iam_role.this[each.key].name
  policy_arn = each.value.policy_arn
}
```

I use this pattern heavily for things like creating per-microservice ECR repos, per-team S3 buckets, or per-environment security groups — anywhere the resource set is naturally keyed rather than ordinal.

---

### Q3. What are Sentinel policies, and how have you used them in practice?
**Answer:**
Sentinel is HashiCorp's policy-as-code framework used with Terraform Cloud/Enterprise to enforce **guardrails before an apply happens** — it runs between the `plan` and `apply` stages.

Policy enforcement levels:
- **advisory** — logs a warning, doesn't block.
- **soft-mandatory** — blocks the apply but can be overridden by an authorized user.
- **hard-mandatory** — blocks unconditionally.

Real examples I've written/enforced:
- Deny any `aws_instance` or `aws_db_instance` without mandatory tags (`CostCenter`, `Owner`, `Environment`).
- Deny public S3 buckets (`acl != "public-read"`, block public access settings must be true).
- Restrict instance types allowed in production to an approved list (cost governance).
- Require encryption-at-rest (`storage_encrypted = true`) on all RDS resources.
- Deny security groups with `0.0.0.0/0` on port 22/3389.

Sentinel policies inspect the `tfplan` import, so they see the **planned state**, not just the HCL — meaning they catch issues even when values come from data sources or modules.

*(Note: If the org uses open-source Terraform instead of TFC/TFE, the equivalent is OPA/Conftest or `tfsec`/`checkov` in the CI pipeline — worth mentioning if asked about alternatives.)*

---

### Q4. How do you structure variable files for multiple environments (dev/stage/prod)?
**Answer:**
Two patterns I've used, depending on team maturity:

**Pattern A — Directory-per-environment (preferred for strong isolation):**
```
environments/
  dev/
    main.tf
    dev.tfvars
    backend.tf
  stage/
    main.tf
    stage.tfvars
    backend.tf
  prod/
    main.tf
    prod.tfvars
    backend.tf
modules/
  vpc/
  eks/
  rds/
```
Each environment has its own state file and backend config — blast radius is contained; a mistake in dev can't touch prod state.

**Pattern B — Workspaces with shared config + per-env `.tfvars`:**
```bash
terraform workspace select prod
terraform apply -var-file="prod.tfvars"
```
Faster to set up, but riskier — a wrong `workspace select` before an apply is a classic production incident. I generally avoid workspaces for prod-critical infra and reserve them for ephemeral/feature-branch environments.

I always keep **secrets out of `.tfvars`** — those go into SSM Parameter Store/Secrets Manager/Vault and are referenced via data sources, with `.tfvars` files committed to Git (non-sensitive values only) and a `.gitignore` for anything like `*.auto.tfvars` that might contain local overrides.

---

### Q5. Walk through your S3 backend setup — with and without DynamoDB.

**Answer:**

**With DynamoDB (traditional locking, pre-Terraform 1.10):**
```hcl
terraform {
  backend "s3" {
    bucket         = "org-terraform-state-prod"
    key            = "eks/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
S3 stores the state file; DynamoDB provides **state locking** via a conditional write on a lock ID, preventing two engineers/pipelines from running `apply` concurrently and corrupting state. The DynamoDB table needs just a primary key `LockID` (String).

**Without DynamoDB (native S3 locking, Terraform 1.10+):**
Terraform added native locking using S3 conditional writes (`use_lockfile = true`), removing the DynamoDB dependency entirely:
```hcl
terraform {
  backend "s3" {
    bucket       = "org-terraform-state-prod"
    key          = "eks/prod/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```
This simplifies infra (one less component to manage/pay for) but requires the S3 bucket to support conditional PUT (which AWS S3 does). I'd mention this as the "modern" approach but note many production environments still run pinned older Terraform versions, so DynamoDB locking remains common in the field.

**Other backend hygiene I enforce:**
- Versioning enabled on the state bucket (recover from accidental corruption).
- Bucket policy restricting access to CI/CD roles and specific IAM principals only.
- Separate state buckets (or at minimum separate keys/prefixes) per environment.
- `encrypt = true` with KMS-backed encryption, not just SSE-S3, for regulated workloads.

---

### Q6. How do you handle Terraform state file conflicts or corruption in a live incident?
**Answer:**
1. First, check if it's a **lock** issue vs **corruption** issue — `terraform force-unlock <LOCK_ID>` only after confirming no other apply is genuinely running.
2. If state is out of sync with reality (e.g., someone manually changed a resource in the console), use `terraform plan` to see the drift, then either `terraform apply` to reconcile, or `terraform import`/`state rm` surgically.
3. If the state file itself is corrupted, restore the last good version from S3 versioning (`aws s3api list-object-versions`), then re-run `plan` to validate.
4. For partial state corruption, `terraform state list`, `state show`, and `state mv` are the surgical tools — never hand-edit the JSON unless absolutely last resort, and even then, back it up first.
5. Long-term fix: this is usually a symptom of too many people running `apply` locally. I push teams toward **CI/CD-only applies** with remote execution (TFC or a runner) so local state manipulation stops being possible.

---

### Q7. How do you integrate Terraform with GitHub Actions for CI/CD?
**Answer:**
Typical pipeline stages:

```yaml
name: terraform-ci
on:
  pull_request:
    paths: ["environments/**"]
  push:
    branches: [main]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -backend-config=backend.hcl
      - run: terraform fmt -check
      - run: terraform validate
      - run: tflint
      - run: checkov -d .
      - run: terraform plan -var-file=prod.tfvars -out=tfplan
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    environment: production   # requires manual approval via GitHub Environments
    steps:
      - uses: actions/download-artifact@v4
      - run: terraform apply tfplan
```

Key design decisions I emphasize:
- **Plan on PR, apply on merge to main** — never apply from a PR branch.
- **GitHub Environments with required reviewers** for prod applies — a real human gate.
- Use **OIDC federation** (`aws-actions/configure-aws-credentials` with `role-to-assume`) instead of long-lived AWS access keys stored as GitHub secrets — this is a common interview follow-up.
- Post the plan output as a PR comment (via `terraform-plan-comment` action or a custom script) so reviewers see the diff before approving.
- Concurrency groups (`concurrency: terraform-prod`) to prevent two applies racing on the same state.

---

### Q8. What's your approach to Terraform module testing?
**Answer:**
- **Static analysis:** `terraform validate`, `tflint`, `checkov`/`tfsec` for security misconfig, `terraform fmt -check` for style.
- **Unit-ish testing:** Terraform's built-in `terraform test` (HCL-based test framework, GA since 1.6) to assert outputs given mock/plan-time inputs.
- **Integration testing:** Terratest (Go-based) — spin up real infra in a sandbox account, assert behavior (e.g., can I actually reach an ALB endpoint), then tear down. Used sparingly since it costs real money/time.
- **Policy testing:** Sentinel/OPA test suites to make sure guardrails behave as expected before rolling to prod.
- I gate merges on static analysis in CI, and run Terratest suites nightly or on module-version-bump PRs rather than every commit, to keep pipelines fast.

---

## Section 2: AWS & Networking

### Q9. How do you design a highly available VPC across multiple AZs?
**Answer:**
Standard 3-tier, 3-AZ design:
- **Public subnets** (one per AZ) — host NAT Gateways and internet-facing ALBs.
- **Private app subnets** (one per AZ) — EKS worker nodes / EC2 app tier, route to internet via NAT Gateway in the *same* AZ (to avoid cross-AZ data transfer charges and reduce blast radius if one AZ's NAT fails).
- **Private data subnets** (one per AZ) — RDS, ElastiCache, isolated with no route to the internet at all.

Key HA decisions:
- **One NAT Gateway per AZ**, not a single shared NAT — otherwise you've built a single point of failure into an "HA" design. (Cost trade-off: 3x NAT Gateway cost vs the risk of full outage if the single AZ's NAT dies.)
- CIDR planning done up front with room to grow — e.g., a `/16` VPC split into `/20` or `/24` subnets per tier per AZ, leaving unused CIDR blocks for future subnets (e.g., a dedicated Lambda/VPC-endpoint subnet later).
- VPC endpoints (Gateway for S3/DynamoDB, Interface for ECR/SSM/Secrets Manager) to keep traffic off the NAT path entirely — cheaper and more secure.
- `terraform-aws-modules/vpc/aws` is the module I typically start from rather than hand-rolling, then customize.

---

### Q10. How do you plan subnetting across AZs for a large, growing environment?
**Answer:**
I plan capacity for at least 2-3 years of growth up front, since VPC CIDR is painful to re-do later:
- Pick a VPC CIDR large enough for all current + future subnets (e.g., `/16` = 65k IPs) even if you'll only use a fraction initially.
- Reserve subnet blocks by **tier and AZ**, not sequentially — e.g., `10.0.0.0/20` = public-AZ-a, `10.0.16.0/20` = public-AZ-b, `10.0.32.0/20` = public-AZ-c, then jump to `10.0.64.0/20` for private-app-AZ-a, etc. Leaving gaps between tiers lets you resize a tier later without renumbering everything.
- For EKS specifically: remember pods consume IPs too if using the AWS VPC CNI (each pod gets an ENI-backed IP by default) — undersized subnets are a common real-world outage cause when node/pod count scales up. I size private app subnets generously (`/19` or larger) for EKS for this reason, or use custom networking / prefix delegation to reduce IP consumption per node.
- Use Secondary CIDR blocks on the VPC if you're retrofitting more IP space into an existing environment without a full re-architecture.

---

### Q11. Explain your experience running EKS in production — what are the non-obvious operational challenges?
**Answer:**
Beyond "just deploy the cluster," the real operational surface area is:
- **Node group strategy:** mix of on-demand (for baseline/critical workloads) and Spot (for stateless/batch) via separate managed node groups or Karpenter for dynamic provisioning. Karpenter has largely replaced Cluster Autoscaler in my recent work — faster bin-packing, better cost efficiency.
- **IRSA (IAM Roles for Service Accounts):** mapping Kubernetes service accounts to fine-grained IAM roles via OIDC, instead of giving broad node-level IAM permissions to every pod on a node.
- **Upgrade strategy:** EKS control plane and node group versions must be upgraded deliberately (one minor version at a time), plus checking add-on compatibility (VPC CNI, CoreDNS, kube-proxy) and deprecated API versions in workloads before upgrading.
- **Networking limits:** IP exhaustion (mentioned above), and security group/ENI limits per instance type affecting max pods per node.
- **Multi-tenancy:** namespace-based isolation with ResourceQuotas/LimitRanges, NetworkPolicies (via Calico or the VPC CNI's native support) to prevent noisy-neighbor and lateral movement issues.
- **Cost visibility:** tagging propagation from Terraform through to Kubernetes labels so Kubecost/CUR-based tooling can attribute spend per team/namespace.

---

### Q12. How do you architect RDS for High Availability and Disaster Recovery?
**Answer:**
Two different problems, two different features:

**High Availability (HA) — Multi-AZ:**
- RDS Multi-AZ deployment maintains a **synchronous** standby replica in a different AZ. On primary failure, RDS automatically fails over (DNS endpoint repoints) — typically 60-120 seconds, no application changes needed.
- Handles AZ-level failures (hardware failure, AZ outage, patching windows) — this is about **uptime within a region**, not disaster recovery.
- For newer workloads I also evaluate Multi-AZ with **two readable standbys** (Aurora-style) or Aurora itself, which has faster failover (sub-30s) and 6-way replication across 3 AZs by design.

**Disaster Recovery (DR) — Read Replicas (cross-region):**
- Cross-region read replicas use **asynchronous** replication — there's replication lag, so this is about surviving a full **regional** outage, not zero-data-loss.
- In a DR event, you **promote** the read replica to a standalone primary (`aws rds promote-read-replica`), which breaks replication permanently — this is a one-way door, so DR runbooks need to be tested, not theoretical.
- I typically pair this with Route 53 health checks (see below) so the app's DB endpoint can be swapped via automation/runbook during failover.

**RPO/RTO framing (interviewers love this):** Multi-AZ gives near-zero RPO and low RTO for AZ failures. Cross-region replicas give a small-but-nonzero RPO (replication lag) and higher RTO (manual/automated promotion + DNS cutover) for regional disasters. I always confirm with the business what RPO/RTO they actually need before over- or under-engineering this.

---

### Q13. How do you use Route 53 health checks for failover?
**Answer:**
- Configure a **Route 53 health check** against an endpoint (HTTP/HTTPS/TCP), pointed at either the app's `/health` endpoint or a load balancer.
- Create a **Failover routing policy**: a PRIMARY record (health-checked) and a SECONDARY record (the DR site). If the primary health check fails (based on configurable failure threshold, e.g., 3 consecutive failures over 30s intervals), Route 53 automatically stops returning the primary's IP and starts resolving to the secondary.
- For active-active setups, I use **Weighted** or **Latency-based** routing combined with health checks instead of strict failover, so traffic naturally shifts away from an unhealthy region rather than an all-or-nothing cutover.
- Important nuance: Route 53 failover is **DNS-based**, so it's subject to DNS TTL/caching — I set low TTLs (e.g., 30-60s) on these records, and make sure clients/resolvers actually respect TTL (some corporate resolvers and mobile carriers cache more aggressively than you'd like — a real gotcha to mention if asked "why didn't failover work instantly").
- Health checks can also monitor CloudWatch alarms directly (not just endpoint pings), useful for triggering failover based on application-level metrics (error rate, latency) rather than pure reachability.

---

## Section 3: Ansible

### Q14. How do you structure playbooks and roles for maintainability at scale?
**Answer:**
Standard `ansible-galaxy`-style role layout:
```
roles/
  webserver/
    tasks/main.yml
    handlers/main.yml
    templates/
    files/
    vars/main.yml
    defaults/main.yml
    meta/main.yml
inventories/
  prod/
    hosts.yml
    group_vars/
    host_vars/
  staging/
    hosts.yml
    group_vars/
site.yml
```
Principles I follow:
- **`defaults/` vs `vars/`:** defaults are low-precedence, overridable by inventory — used for anything environment-specific. `vars` are role-internal constants that shouldn't be overridden casually.
- One role = one responsibility (e.g., `nginx`, `docker`, `node_exporter`) — composed together in a top-level playbook rather than one giant role doing everything.
- `group_vars`/`host_vars` per environment for anything that differs between on-prem and cloud, or dev/stage/prod.
- Idempotency is non-negotiable — every task should be safely re-runnable; I avoid raw `shell`/`command` modules unless there's genuinely no proper module, and when I must use them, I add `creates`/`changed_when`/`failed_when` guards.
- Tags on tasks/roles (`--tags patching`, `--tags deploy`) so a full site run isn't required for targeted changes.

---

### Q15. Give an example of a Jinja2 template you'd use and why templating matters here.
**Answer:**
Example — templating an nginx config that differs per environment:
```jinja
# templates/nginx.conf.j2
upstream app_backend {
{% for host in groups['app_servers'] %}
    server {{ hostvars[host]['ansible_host'] }}:{{ app_port | default(8080) }};
{% endfor %}
}

server {
    listen 80;
    server_name {{ server_name }};
    {% if enable_ssl | default(false) %}
    listen 443 ssl;
    ssl_certificate {{ ssl_cert_path }};
    ssl_certificate_key {{ ssl_key_path }};
    {% endif %}
}
```
Applied via:
```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/app.conf
    owner: root
    mode: '0644'
  notify: reload nginx
```
Why this matters at scale: instead of maintaining N static config files for N environments/hosts, one template plus inventory variables (`group_vars/prod.yml`, `group_vars/staging.yml`) generates the correct output per target — the `groups['app_servers']` loop even dynamically reflects however many backend hosts exist in inventory at run time, so scaling the fleet doesn't require touching the template.

---

### Q16. When and how have you written custom Ansible modules in Python?
**Answer:**
I reach for a custom module when: no existing module covers an internal/proprietary system (e.g., a company-internal CMDB, a legacy on-prem appliance's API, a niche network device), or when a sequence of shell tasks needs proper idempotency/check-mode support that raw shell can't give cleanly.

Structure of a minimal custom module:
```python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule

def run_module():
    module_args = dict(
        name=dict(type='str', required=True),
        state=dict(type='str', default='present', choices=['present', 'absent']),
    )
    result = dict(changed=False, name='')

    module = AnsibleModule(argument_spec=module_args, supports_check_mode=True)
    name = module.params['name']
    state = module.params['state']

    current = check_current_state(name)   # custom logic
    if state == 'present' and not current:
        if not module.check_mode:
            create_resource(name)
        result['changed'] = True
    elif state == 'absent' and current:
        if not module.check_mode:
            delete_resource(name)
        result['changed'] = True

    result['name'] = name
    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
```
Key things I make sure custom modules get right (these are common interview probes):
- **`supports_check_mode=True`** and honoring `--check` — critical for teams that run dry-runs before real changes.
- Proper `changed` reporting — false "changed" status breaks idempotency reporting and confuses drift detection.
- Modules go in `library/` at the playbook root, or in a proper collection (`ansible-collections` structure) if shared across projects — I prefer building an internal **Ansible Collection** once more than one or two modules exist, for versioning and distribution via `ansible-galaxy collection install`.

---

### Q17. How do you manage Ansible for hybrid on-prem + cloud environments?
**Answer:**
- **Dynamic inventory** for cloud (`aws_ec2` inventory plugin pulling live EC2 instances by tag) combined with **static inventory** (`hosts.ini`/`hosts.yml`) for on-prem, often merged via multiple inventory sources passed to `ansible-playbook -i aws_ec2.yml -i onprem_hosts.yml`.
- Group on-prem and cloud hosts by **function**, not by location, in `group_vars` (e.g., `webservers` group spans both, with location-specific overrides in `host_vars` only where truly needed) — this keeps playbooks environment-agnostic.
- Connection differences: on-prem often needs `ansible_ssh_common_args` for jump hosts/bastions; cloud instances often use SSM Session Manager as the connection plugin (`community.aws.aws_ssm`) instead of direct SSH, especially for private-subnet EC2 instances with no bastion.
- Secrets: Ansible Vault for on-prem-only secrets, but for cloud I prefer pulling secrets at runtime from AWS Secrets Manager/SSM Parameter Store via lookup plugins, so nothing sensitive sits encrypted-but-static in Git.
- Patching cadence differs — on-prem often has fixed maintenance windows (`serial:` keyword to control batch size and avoid taking down all app servers at once), while cloud can lean more on blue/green replacement via ASG instance refresh instead of in-place patching.

---

## Section 4: GitOps & CI/CD

### Q18. Explain your GitOps workflow with Argo CD — how does it differ from a traditional push-based CD pipeline?
**Answer:**
Traditional CI/CD is **push-based**: the pipeline (e.g., GitHub Actions) has credentials to the cluster and runs `kubectl apply`/`helm upgrade` directly against it.

GitOps with Argo CD is **pull-based**: Argo CD runs *inside* the cluster (or has cluster credentials scoped to it) and continuously reconciles the cluster's actual state against the desired state declared in a Git repo. CI never touches the cluster directly — it only updates a manifest/Helm values file in Git (e.g., bumping an image tag), and Argo CD detects the diff and syncs.

Why this matters:
- **Security:** no cluster credentials need to live in CI/CD systems — smaller attack surface.
- **Auditability:** Git history *is* the deployment history — every change to running state is a traceable commit.
- **Drift detection/self-healing:** if someone manually `kubectl edit`s something in the cluster, Argo CD flags it as OutOfSync and (if auto-sync + self-heal is enabled) reverts it back to match Git — enforcing Git as the single source of truth.
- **Rollback = `git revert`**, not re-running a deploy pipeline against old artifacts.

Typical flow I've implemented: CI (GitHub Actions) builds/tests/pushes the image to ECR, then updates the image tag in a separate **GitOps config repo** (not the app repo) via a bot commit or PR; Argo CD watches that repo and syncs.

---

### Q19. What are Argo CD ApplicationSets and when would you use them?
**Answer:**
A single Argo CD `Application` maps one Git source to one cluster/namespace target. That doesn't scale when you have, say, 20 microservices × 3 environments × multiple clusters — you'd be hand-maintaining dozens of near-identical Application manifests.

`ApplicationSet` solves this by **generating** Applications dynamically from a generator:
- **List generator** — explicit list of environments/clusters.
- **Cluster generator** — auto-discovers every cluster registered to Argo CD and deploys the same app to all of them (great for a fleet of edge/regional clusters).
- **Git generator (directory or file-based)** — scans a repo structure (e.g., `apps/*/values-{env}.yaml`) and creates one Application per matched directory/file — this is what I use most for "one app, many environments" patterns.
- **Matrix generator** — combines two generators (e.g., cross product of services × clusters).

Example use case from my experience: a monorepo with `apps/<service>/overlays/<env>/` Kustomize structure — a single ApplicationSet with a Git directory generator automatically creates/removes Argo CD Applications as services or environments are added/removed, with zero manual Argo CD config changes.

---

### Q20. How do you structure environment branching with a Git flow strategy for CI/CD?
**Answer:**
I'm generally cautious about classic Git Flow (long-lived `develop`/`release`/`feature` branches) for CI/CD-heavy teams — it tends to create merge pain and slows down deploy velocity. What I've actually implemented successfully at scale:

**Trunk-based + environment promotion via GitOps repo (preferred):**
- Single `main` branch for application code, short-lived feature branches merged via PR with required checks.
- Environment state lives in the **GitOps repo**, not app branches — e.g., `environments/dev/values.yaml`, `environments/stage/values.yaml`, `environments/prod/values.yaml`.
- Promotion = a PR that bumps the image tag/values from `dev` → `stage` → `prod` folders, reviewed and merged like code. This gives an auditable, gated promotion path without needing long-lived app branches.

**When true Git Flow / branch-per-environment is still used (e.g., legacy or compliance-driven orgs):**
- `develop` → auto-deploys to dev on every merge.
- `release/*` branches → deploy to staging for QA sign-off.
- `main`/`master` → tagged releases deploy to prod, often with manual approval gates in GitHub Actions Environments.
- Hotfix branches off `main` for emergency prod patches, merged back into `develop` to avoid drift.

I explain the trade-off explicitly in interviews: branch-per-environment makes "what's in prod right now" a Git branch question, while GitOps-repo-based promotion makes it a **folder/PR** question — the latter scales much better with microservices and reduces merge conflict overhead, which is why most GitOps-mature orgs (mine included) moved away from strict Git Flow for deployment purposes, while still using lightweight trunk-based flow for application source code.

---

### Q21. How do you handle secrets in a GitOps model, where "everything is in Git"?
**Answer:**
Never plaintext in Git — a few approaches I've used, chosen based on the maturity of the org:
- **Sealed Secrets** (Bitnami) — encrypt secrets client-side with a public key; only the in-cluster controller (holding the private key) can decrypt. Safe to commit the sealed/encrypted blob to Git.
- **SOPS + KMS/age** — encrypt YAML/JSON values in place using AWS KMS, decrypted at sync time via a plugin (e.g., `argocd-vault-plugin` or a KSOPS-enabled Argo CD).
- **External Secrets Operator** — the GitOps repo only references a secret *name/path*, and ESO pulls the actual value at runtime from AWS Secrets Manager/SSM/Vault, so no secret material ever touches Git at all — this is my preferred pattern for AWS-heavy environments, since it centralizes secret rotation in Secrets Manager rather than needing to re-encrypt/re-commit on every rotation.

---

### Q22. Describe a production incident you handled involving Terraform, Kubernetes, or CI/CD, and what you changed afterward.
**Answer (framework to use — tailor with a real example in the actual interview):**
Use the **STAR** method (Situation, Task, Action, Result) and pick a story that demonstrates:
1. **Diagnosis under pressure** — e.g., an Argo CD auto-sync + self-heal fighting against a manual emergency `kubectl` fix during an incident, causing a flapping rollback loop.
2. **Root cause, not just symptom fix** — e.g., realizing the real issue was no `PreSync` hook / no maintenance-mode toggle for genuine emergency overrides.
3. **Concrete process/tooling change afterward** — e.g., added an documented "pause auto-sync" runbook step, added Sentinel/OPA policy to prevent the misconfiguration that caused it, or added a canary/progressive-delivery step (Argo Rollouts) so the blast radius of the next similar issue is automatically contained.

Interviewers at this level are specifically listening for **ownership of the process improvement**, not just "I fixed it" — always close incident stories with "and here's what changed structurally so it can't happen the same way again."

---

## Quick-Fire Round (rapid depth-check questions interviewers often use as tie-breakers)

| Question | What a strong 16+ yr answer signals |
|---|---|
| `terraform plan` shows a resource will be destroyed and recreated — how do you avoid downtime? | Knows `create_before_destroy` lifecycle, or restructuring to avoid forced replacement (e.g., renaming triggers replacement on some resources) |
| Difference between `count` and `for_each` when a list has duplicate values? | Knows `for_each` requires unique keys; duplicate values in a list passed via `toset()` will error/collapse |
| How do you avoid Ansible playbooks silently failing mid-run across 200 hosts? | Knows `serial`, `max_fail_percentage`, `any_errors_fatal`, and proper `--check`/`--diff` dry runs before real runs |
| Why might Argo CD show "OutOfSync" even right after a successful sync? | Knows about non-deterministic fields (e.g., mutating webhooks, HPA-managed replica counts) needing `ignoreDifferences` config |
| RDS Multi-AZ failover just happened — what changes for the app? | Knows the endpoint DNS doesn't change, but existing connections drop and must reconnect — app-level connection retry/pooling matters |
| Why prefer OIDC over static AWS keys in GitHub Actions? | Knows short-lived, auto-rotated tokens vs long-lived secrets sitting in GitHub Secrets as a standing liability |

---

*Tip for the actual interview: at 16+ years, interviewers are testing for trade-off reasoning and incident scars, not memorized syntax. For every answer, be ready to say "the alternative I considered was X, and I chose Y because..." — that's what separates a senior/staff-level answer from a mid-level one.*