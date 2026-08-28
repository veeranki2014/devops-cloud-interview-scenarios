# Kubernetes SME Interview Prep
**Role:** Kubernetes SME | Pittsburgh, PA (Onsite) | Contract-to-Hire
**Candidate:** Ankarao Veeranki — 16+ yrs, CKA + RHCE, Principal Platform Engineer @ Freddie Mac

---

## 1. JD Breakdown — What They're Actually Screening For

| JD Requirement | What it really means | Your evidence |
|---|---|---|
| 3–5 yrs Kubernetes-focused experience | Hands-on cluster ops, not just "used K8s" | Freddie Mac (2023–present) + Mashreq Bank (2013–2023) K8s work — you exceed this significantly |
| 6–9 yrs Platform Engineering | Owning infra as a product for other teams, not just app deployment | Both Freddie Mac and Mashreq roles are platform-team roles supporting multiple app teams |
| IaC | Terraform module design, drift management, reusable patterns | Terraform, CloudFormation, Ansible, Helm — reusable modules for EKS/IAM/RDS/S3/networking |
| Cloud Native Architectures | Microservices, service mesh, container runtimes, resiliency patterns | Istio service mesh, CNCF tooling (Prometheus, Grafana, Fluentd, OPA, Cert-Manager) |

**"SME" + "Contract to Hire" signals**: They want someone who can be the technical escalation point on day one (you've done this at Freddie Mac) and who they can evaluate for a few months before committing — expect practical/scenario-heavy interviews over theory.

---

## 2. Core Areas to Prepare (in priority order)

### A. Kubernetes Cluster Architecture & Lifecycle (highest weight for "SME")
- Control plane components (API server, etcd, scheduler, controller-manager) and failure modes for each.
- EKS-specific: managed control plane vs self-managed nodes, node groups vs Fargate, cluster upgrade process (control plane → node groups, version skew policy).
- Upgrade/patching strategy: blue-green node groups, surge upgrades, PodDisruptionBudgets during upgrades.
- Multi-tenancy: namespace isolation, ResourceQuotas/LimitRanges, RBAC design, NetworkPolicies, Pod Security Standards (replacing PSPs).
- **Be ready to whiteboard**: "Design a multi-tenant EKS platform for 5 business units with isolation and quota guarantees."

### B. Networking & Storage (CNI/CSI) — SME-level troubleshooting
- CNI: how pod networking works (VPC CNI on EKS specifically — ENI/IP exhaustion is a classic real-world EKS pain point), how to debug `CrashLoopBackOff` vs `ImagePullBackOff` vs networking-caused pod failures.
- Service types (ClusterIP/NodePort/LoadBalancer), Ingress controllers, and when to use a service mesh (Istio) instead of/alongside Ingress.
- CSI: PV/PVC lifecycle, StorageClasses, EBS/EFS CSI driver specifics on EKS, StatefulSet storage considerations.
- **Prepare a real incident story**: you list "Troubleshoot complex cluster, networking (CNI), storage (CSI) at SME level" — have one concrete example ready (symptom → diagnosis commands → root cause → fix).

### C. IaC & GitOps
- Terraform: module design for reusability, state management/locking, drift detection & remediation (you list this explicitly — expect a direct question: "how do you detect and remediate drift?").
- Helm vs Kustomize — when you'd choose one over the other, and how you structure overlays across Dev/QA/Prod.
- ArgoCD: App-of-Apps pattern, sync policies (auto vs manual), rollback strategy, secrets handling in GitOps (avoid secrets in Git — Sealed Secrets/External Secrets Operator/Vault).
- **Be ready to explain your GitOps pipeline end-to-end**: commit → CI (build/scan/test) → image push → ArgoCD sync → deployment → verification.

### D. Cloud-Native Architecture & Resiliency
- Service mesh (Istio) — why you'd introduce it (mTLS, traffic shaping, canary/blue-green, circuit breaking) vs the complexity cost.
- Multi-region/DR patterns for stateless vs stateful workloads on EKS.
- Autoscaling: HPA vs VPA vs Cluster Autoscaler vs Karpenter (Karpenter is increasingly what interviewers probe for since it's replacing Cluster Autoscaler in a lot of shops — worth a quick refresh even if not on your resume).
- Resiliency patterns: PodDisruptionBudgets, topology spread constraints, anti-affinity for AZ resilience.

### E. Security / DevSecOps
- Image scanning (Trivy — on your resume), SonarQube gates, admission control with OPA/Gatekeeper.
- Secrets management: Cert-Manager for TLS automation, Kubernetes Secrets vs external secret stores.
- RBAC least-privilege design, Pod Security Standards enforcement.
- IRSA (IAM Roles for Service Accounts) on EKS — very likely to come up given your AWS-heavy background.

### F. Observability & SLO Ownership
- You explicitly list "establish SLIs/SLOs" — have concrete numbers ready (e.g., "99.9% availability target, alerted at 2% error-budget burn").
- Prometheus/Grafana stack: what you scrape, key K8s metrics (node pressure, pod restarts, OOMKilled events, API server latency), alerting via Alertmanager.
- Logging: Fluentd → centralized logging, correlation during incident response.

### G. CI/CD Integration
- Jenkins vs GitHub Actions — when each is used in your environments, pipeline stages (build → test → scan → push → deploy).
- How CI hands off to CD (ArgoCD) — separation of concerns between the two.

---

## 3. Likely Behavioral / SME-Framing Questions (STAR-ready, using your resume)

**"Tell me about a time you were the senior escalation point for a platform outage."**
→ Use your Freddie Mac bullet: "senior-most technical escalation point... mentoring junior platform engineers." Structure as: Situation (prod outage/incident), Task (you as SME), Action (RCA process, war-room coordination), Result (fix + postmortem + prevention).

**"How have you driven IaC governance across teams?"**
→ Use: reusable Terraform modules (networking, EKS, IAM, RDS, S3), drift detection/remediation, version control practices. Explain the *before/after* — what was inconsistent before, what standard you enforced.

**"Describe a Kubernetes migration you led."**
→ Use Mashreq Bank: "Guided application teams through containerization, K8s migration, deployment modernization, production-readiness reviews." Prepare a specific app example: what blocked it (stateful data? legacy config?), how you resolved it, readiness checklist you used.

**"How do you balance being hands-on vs enabling other teams (platform-as-a-product mindset)?"**
→ This is core to "Platform Engineering" — talk about self-service via Helm charts/Terraform modules, golden paths, and reducing tickets by building reusable patterns instead of doing one-off work per team.

**"Walk me through your incident response process."**
→ Detection (alerting) → triage/roles → mitigation → RCA → postmortem → follow-up action items. Mention a real Kubernetes-specific example (node pressure eviction storm, bad rollout via ArgoCD needing rollback, etc.).

---

## 4. Scenario / Whiteboard Questions to Rehearse Out Loud

1. "A pod is stuck in `Pending` — walk me through your debugging steps." *(Answer path: `kubectl describe pod` → check events → resource requests vs node capacity → taints/tolerations → PVC binding issues → scheduler logs)*
2. "Nodes are hitting IP exhaustion on EKS — how do you fix it?" *(VPC CNI prefix delegation, secondary CIDR, right-sizing subnets)*
3. "A deployment via ArgoCD is stuck in `OutOfSync` — how do you resolve without breaking prod?" *(diff review, manual sync with pruning caution, resource hooks, rollback via git revert)*
4. "How would you design zero-downtime cluster upgrades for a Tier-1 workload?" *(PDBs, surge node groups, draining strategy, canary validation before full rollout)*
5. "How do you enforce that no pod runs as root or with a privileged container across 50 namespaces?" *(Pod Security Standards / OPA Gatekeeper policies, admission webhooks, CI-side scanning as a second gate)*

---

## 5. Gaps to Shore Up Before the Interview

Your resume is strong on AWS/EKS-native tooling. A few areas worth a quick refresh since "SME" interviews probe breadth:
- **Karpenter** vs Cluster Autoscaler (newer AWS-recommended pattern) — mention familiarity even if you've mainly used Cluster Autoscaler/ASGs.
- **Kubernetes API deprecations/version skew policy** — be ready to name what changed in the last 2–3 minor versions (e.g., PSP removal, in-tree to CSI driver migrations) since this signals you stay current.
- **Cost optimization** — Spot instances with Karpenter/node groups, right-sizing via VPA recommendations — increasingly a standard SME topic given this sounds like a cost-conscious mid-size platform team.

---

## 6. Questions to Ask Them (shows SME-level thinking, not just candidate-level)

- "What does the current on-call/escalation model look like, and where does this role sit in it?"
- "Is the platform multi-cluster/multi-region today, or is that on the roadmap?"
- "What's driving the need for a contract SME right now — a migration, an incident history, or scaling ahead of growth?"
- "What does 'Kubernetes Delivery platform' mean here specifically — is it internal developer platform work, or more infra/SRE-focused?"

---

*Given this is Contract-to-Hire, expect the technical bar to be high but the culture-fit bar to matter too — they're evaluating a few months of working relationship, not just a one-time hire. Lean into the "mentoring junior engineers" and "cross-team standards" angle from your Freddie Mac role — that's exactly what a CTH SME hire needs to demonstrate early.*