# Interview Guide — Container & VM Software Engineer (Platform Engineering)
### Candidate Profile: ~16 Years Experience (Senior / Lead Level)

This guide is designed for a candidate at architect/lead maturity — expect answers that go beyond "how to" and into **why, trade-offs, scale, and how they've led others through it.** Questions are grouped by JD theme, roughly in interview order: platform fundamentals → automation depth → lifecycle/architecture → AI-agent enablement → security → SRE/leadership → behavioral.

---

## 1. OpenShift/Kubernetes & Linux VM Administration

**Q1. You're inheriting a mixed estate — some workloads on OpenShift, some on Linux VMs that haven't been containerized yet. How do you decide what to migrate first, and what stays on VMs long-term?**

*Model Answer:* At 16 years, the answer should show a prioritization framework, not just tooling. Look at:
- **Statefulness and I/O pattern** — stateless, horizontally scalable services move first; workloads with tight kernel dependencies, licensing constraints, or exotic hardware/driver needs (e.g., some COTS apps) may never be good container candidates.
- **Blast radius and business criticality** — start with low-risk, high-repetition workloads to build migration muscle memory and prove the playbook before touching Tier-0 systems.
- **Cost of the "do nothing" path** — VMs with high patch/CVE churn or approaching EOL OS versions get prioritized regardless of complexity, because the risk of inaction is higher than the migration effort.
- A candidate at this level should also mention building a **scoring rubric** (dependency count, statefulness, compliance tier, team readiness) so the decision isn't ad hoc, and that some VMs are intentionally left as VMs with the *VM Image Factory* pattern applied instead — repaved on the same cadence as containers, so the platform's reliability guarantee doesn't depend on 100% containerization.

**Q2. Describe how you'd design a repave strategy for an OpenShift cluster that cannot tolerate downtime, versus one for a fleet of stateless VMs.**

*Model Answer:*
- **OpenShift/Kubernetes:** Rolling node replacement using cordon → drain → terminate → replace with new golden image, respecting PodDisruptionBudgets, topology spread constraints, and surge/unavailable settings on MachineSets. For stateful workloads, ensure PVs are backed by a storage class that supports live migration or that StatefulSets have proper anti-affinity so quorum isn't lost mid-repave.
- **VM fleet:** Blue/green at the load-balancer or service-discovery layer — bring up new-image VMs, register them, shift traffic gradually (canary %), monitor health, then deregister and destroy the old fleet. This is where **VM image lineage tracking and service-discovery tooling** (as called out in the JD) becomes critical: you need to know which image version each VM instance is running and be able to correlate that with traffic health in real time.
- Key point to listen for: the candidate should tie this back to **idempotent automation** — the repave operation should be safely re-runnable if it's interrupted partway.

**Q3. Walk me through how you'd design an OpenShift cluster's compliance/security posture for a regulated enterprise environment (SOC2/PCI/FedRAMP-adjacent).**

*Model Answer:* Should touch: SCC (Security Context Constraints) hardening and default deny, namespace-level RBAC least privilege, network policies (default-deny ingress/egress with explicit allow), image provenance/signing (cosign/sigstore), admission control (OPA/Gatekeeper or Kyverno) to block non-compliant image sources, and integration with a central vulnerability scanner gating the CI pipeline before promotion. A senior candidate should also mention **audit logging retention and immutability** and periodic **CIS benchmark** scans as ongoing compliance, not one-time setup.

---

## 2. Ansible & Packer — Configuration and Image Automation

**Q4. How do you structure a large Ansible codebase (roles, inventories, variables) so that 50+ engineers across multiple teams can contribute without stepping on each other or introducing drift?**

*Model Answer:* Expect discussion of:
- Role-based decomposition with a clear **contract** (input vars, tags, idempotency guarantees) per role, published/versioned like a library (e.g., via Ansible Galaxy or a private equivalent).
- Dynamic inventories sourced from CMDB/cloud provider rather than static files, to avoid inventory drift.
- **Molecule** or equivalent for testing roles in isolation before merge.
- A clear separation between "platform" playbooks (owned centrally) and "application" playbooks (owned by app teams) with a defined interface, so platform changes don't silently break app-team automation.
- Enforcing idempotency via CI — every playbook run twice in a row should produce zero changes on the second run; that's a gating check, not a guideline.

**Q5. Packer builds are slow and your team's golden images take 40 minutes to build, blocking rapid CVE-patch turnaround. How do you speed this up?**

*Model Answer:* Layered image strategy — separate the rarely-changing base layer (OS + hardening) from the frequently-changing layer (latest patches, app runtime). Use a caching build pipeline (e.g., pre-baked base AMI/qcow2 that only the patch layer rebuilds against), parallelize provisioners where independent, and consider moving some provisioning from Packer's shell/Ansible provisioner into a **multi-stage container build** pattern for the container-image path specifically. Also worth mentioning: decoupling **build** from **promotion** — build continuously in the background, but only promote/sign the image that passes scanning, so the critical path for a security fix is "promote existing scanned image" not "rebuild from scratch."

**Q6. How do you validate that a new golden image is safe to roll out fleet-wide before you repave production?**

*Model Answer:* A staged promotion pipeline: build → automated smoke tests (boot, health checks, basic app functionality) → vulnerability scan gate → canary deployment to a small % of non-critical hosts/pods → automated rollback trigger on error-rate/latency regression → progressive rollout to remaining fleet. Should mention **image lineage/tagging** so any regression can be traced back to exactly which image version and which Ansible/Packer commit produced it.

---

## 3. CI/CD Pipeline Engineering

**Q7. Design a CI/CD pipeline that builds, scans, and deploys both container images and VM images from the same source repo, with different promotion gates for each.**

*Model Answer:* Look for a pipeline-as-code answer (Jenkins/GitLab CI/Tekton) with:
- Shared early stages (lint, unit test, SAST) regardless of target.
- Divergent build stage: `docker/buildah build` + push to registry vs. `packer build` + image registration in cloud provider/VM image store.
- Both paths converge again at a **scan gate** (Trivy/Grype/Twistlock or equivalent) with a shared severity policy, so container and VM images are held to the same security bar.
- Environment promotion via GitOps (ArgoCD/Flux) for containers, and a controlled rollout tool (e.g., Ansible Tower/AWX job templates, or a custom orchestrator) for VM fleets — the candidate should acknowledge these are *different mechanisms* converging on the same policy, not force-fit into one tool.
- Mention of **pipeline observability** — every build/deploy emits structured events to the same observability stack, so a platform engineer doesn't need six different dashboards to know fleet health.

**Q8. A pipeline that used to take 8 minutes now takes 35 minutes as the org scaled. How do you diagnose and fix that without just throwing more compute at it?**

*Model Answer:* Systematic: profile stage-by-stage timing first (most teams skip this and guess). Common culprits at scale: serialized test suites that should be parallelized/sharded, monolithic pipelines that rebuild everything on every commit instead of using change detection/path filters, registry/artifact pull bottlenecks, and shared runners under contention. Fix priority: parallelize before scaling hardware, cache aggressively (dependency caches, layer caches), and split monorepo pipelines by affected component.

---

## 4. Image & VM Lifecycle Ownership (Core to this JD)

**Q9. The JD emphasizes "repave, rebuild, patch, and retire on a defined cadence." How would you define that cadence, and what tension exists between cadence and application stability?**

*Model Answer:* This is a strong signal question for seniority. Expect:
- Cadence should be **risk-tiered**, not one-size-fits-all: e.g., weekly patch-only repaves for low-risk services, monthly full rebuilds for standard tier, and a faster out-of-band path for critical CVEs (same-day/48-hour SLA) regardless of tier.
- The core tension: application teams want stability and predictable release windows; the platform team wants zero drift. Resolution is usually a **published, opt-out-with-justification model** — repaves happen automatically on schedule unless an app team has an approved, time-boxed exception (which itself becomes a tracked risk item, not a silent gap).
- A senior candidate should mention this requires **organizational buy-in**, not just tooling — they've likely had to negotiate this with app teams before and can describe how they got adoption (e.g., piloting on a friendly team, showing MTTR/vulnerability-window improvements as data).

**Q10. How do you build "near-zero manually managed exceptions" as stated in the success metrics — practically, what does an exception look like and how do you drive it down over time?**

*Model Answer:* An exception is typically a workload that can't be auto-repaved due to: a hard dependency on a specific patch level, a legacy app with no health-check endpoint (so the platform can't safely validate post-repave health), or an app owner who hasn't onboarded to the automation yet. Driving it down requires: a **visible exception registry/dashboard** (not hidden in tickets), an aging SLA on exceptions (e.g., flagged for escalation after 30/60/90 days), and treating exception remediation as its own workstream with headcount/story points allocated — not something squeezed in around "real" project work.

**Q11. Explain your approach to VM image lineage tracking. Why does it matter, and how would you implement it?**

*Model Answer:* Lineage tracking means every running instance (VM or pod) can be traced back to: the exact golden image ID/hash, the Packer/Ansible commit that built it, the base OS patch level, and the timestamp of last repave. Implementation: tag every image at build time with immutable metadata (git SHA, build timestamp, CVE scan report ID), persist that in a queryable store (could be as simple as a database table keyed by image ID, or integrated into the CMDB), and expose it via a lightweight API/UI so both platform and app teams can self-serve "what's running where, and how old is it." This directly enables the JD's mention of **intelligent traffic shifting during repave cutovers** — you can't safely shift traffic away from an old image version if you don't know which instances are running it.

---

## 5. Full-Stack Platform Tooling

**Q12. The JD mentions building platform tooling with Java/.NET/Go/Python + React/Angular, backed by Oracle/SQL. Give an example of a tool you'd build for this role, and how you'd architect it.**

*Model Answer:* Listen for a concrete example relevant to this JD — e.g., a **self-service repave dashboard** where app owners can see their fleet's drift status, request an exception, or trigger an on-demand repave. Architecture should reflect microservices thinking: a backend API (Go/Java) fronting the image lineage DB and orchestration triggers, a thin React/Angular UI, and clear separation between the **control plane** (this tool) and the **data plane** (actual Ansible/Packer/K8s operations) — the tool should orchestrate, not directly execute privileged operations, for security/blast-radius reasons.

**Q13. How do you decide when a platform capability needs a UI at all, versus just being a CLI/API/GitOps-driven workflow?**

*Model Answer:* A senior answer should reflect pragmatism: UIs are worth building when the audience is broad and non-specialist (e.g., app team leads who don't live in the terminal) or when visibility/dashboarding itself is the value (drift status, lineage). Pure platform engineers and automation generally prefer API/CLI/GitOps because it's scriptable and auditable via version control. Over-building UI for internal tooling is a common trap the candidate should show awareness of avoiding.

---

## 6. AI Coding Agents (Devin or equivalent) — Newer JD Theme

**Q14. The JD calls out using AI coding agents to accelerate Category-2/3 application onboarding. How do you scope a task so an AI agent can work on it safely and productively, versus a task you'd keep fully human-owned?**

*Model Answer:* Good candidates will distinguish by **risk and ambiguity**:
- Good AI-agent fit: well-bounded, pattern-repeatable work — writing Dockerfiles for a known app stack, generating Ansible role boilerplate from a template, updating dependency versions, writing initial CI pipeline YAML, migrating simple COTS configs.
- Poor fit / human-owned: anything touching production credentials/secrets directly, architecture decisions with long-term cost, security policy changes, and Tier-0 workload cutovers.
- Process maturity signal: the candidate should describe a **review gate** — AI-agent output goes through the same PR/scan/test pipeline as human-written code, with no special bypass, and someone senior reviews the diff. They should also mention giving the agent a **narrow, well-defined task with acceptance criteria** (e.g., "containerize this app such that it passes these three health checks") rather than open-ended instructions, because that's where agent reliability breaks down.

**Q15. What risks or failure modes have you seen (or would you anticipate) when scaling AI-assisted migration work across many application teams, and how do you mitigate them?**

*Model Answer:* Expect: inconsistent output quality across different app stacks (agent may confidently produce plausible-but-wrong config for an unfamiliar framework), secret/credential leakage if the agent is given broader repo access than needed, and "review fatigue" where humans start rubber-stamping AI-generated PRs as volume increases. Mitigations: standardized onboarding templates/patterns the agent is grounded against, scoped/least-privilege access per task, and treating AI-generated PRs with the *same* or *stricter* review bar, potentially with automated policy checks (OPA/Kyverno, linting) doing the first pass so humans review by exception.

---

## 7. Security & Hardened Base Images

**Q16. Walk me through how you'd migrate an application fleet from standard base images (e.g., Ubuntu/RHEL full) to minimal/hardened images like Chainguard or Red Hat UBI, without breaking things.**

*Model Answer:* Should mention: auditing actual runtime dependencies first (many apps carry unused packages/shells/package managers that minimal images strip out), staged rollout starting with stateless/low-risk services, working with app teams to fix hard-coded assumptions (e.g., debugging via `bash` inside a distroless container isn't possible — need ephemeral debug containers instead), and measuring the CVE-count reduction as a concrete win to build momentum for wider adoption. A senior candidate will flag the cultural change management piece — dev teams often resist minimal images because it removes debugging conveniences, and part of the job is making the new workflow (e.g., `kubectl debug`) as good or better.

**Q17. How do you keep vulnerability remediation SLAs (partnering with security teams) without turning platform engineering into a full-time patch-firefighting function?**

*Model Answer:* Automation is the lever: continuous scanning integrated into CI (fail builds on critical CVEs above policy threshold), automatic minor-version base image bumps via bots (like Renovate/Dependabot-equivalent for images), and the repave cadence itself absorbing most patching so it's routine, not exceptional. Firefighting should be reserved for zero-days/critical out-of-band CVEs; everything else rides the standard cadence.

---

## 8. SRE, On-Call, and Operational Handoff

**Q18. Describe how you've structured a project-to-BAU (business-as-usual) handoff for a newly onboarded workload. What goes wrong when this is done poorly?**

*Model Answer:* A strong candidate describes a **checklist-driven handoff**: runbooks exist and have been validated (not just written), on-call rotation includes the receiving team with a shadow period, dashboards/alerts are tuned to avoid noise before handoff (alert fatigue is the #1 reason handoffs fail), and there's a defined post-handoff support window where the delivery team remains reachable for escalations. Failure mode to listen for: teams that hand off "done" software with no operational tooling, leaving BAU/SRE teams to reverse-engineer how to operate it — this is exactly the kind of gap 16 years of experience should have hard-won lessons about.

**Q19. Tell me about an incident where drift (config, image, or infra) caused a production issue. What was the root cause and what did you change afterward?**

*Model Answer:* (Behavioral/STAR — listen for specifics, not generic answers.) Should include: how drift was detected (or *not* detected until impact — often the honest and more useful story), what the immediate fix was, and critically, the **systemic fix** — e.g., adding drift-detection tooling, tightening the repave cadence, or adding a pre-deploy config diff check. A candidate at this level should show they think in terms of "how do I make this class of incident structurally impossible," not just "how did I fix this one."

---

## 9. Leadership, Architecture Judgment & Behavioral

**Q20. You're leading a migration engagement and the application team's timeline conflicts with the platform's repave cadence — they want to freeze changes before a big release, but their VMs are due for a security repave. How do you resolve this?**

*Model Answer:* Listen for negotiation and risk communication skills, not just a technical answer: quantify the actual risk of delaying (what CVEs are outstanding, severity), offer a time-boxed, documented exception with a hard end date, and make sure the exception is visible in the tracking system mentioned in Q10 — not a quiet side deal. If the CVE is critical, they should be willing to push back and explain why the repave can't wait, escalating to a joint decision with security/leadership rather than unilaterally overriding the app team.

**Q21. Give an example of a technical decision you made that you'd make differently today, with the benefit of hindsight.**

*Model Answer:* Open-ended — the real signal here is whether the candidate can be honestly self-critical (a trait that matters a lot at platform-engineering scale, where mistakes replicate across the whole fleet) and whether they can articulate *what changed in their thinking*, not just "we picked the wrong tool."

**Q22. How do you mentor or uplift a team that's used to manual, ad hoc operations into an automation-first, self-healing platform mindset?**

*Model Answer:* Expect discussion of incremental wins (automate the most painful manual task first to build trust), pairing/shadowing during early automated repaves so the team sees it work before being asked to rely on it, and shifting the team's metrics/incentives away from "tickets closed" toward "drift eliminated" or "% of fleet auto-managed" so behavior follows the new goal.

**Q23. With 16 years of experience, you've likely seen multiple "waves" of infrastructure paradigms (bare metal → VM → containers → now AI-assisted ops). How do you evaluate whether a new tool or paradigm (like AI coding agents) is worth adopting versus hype?**

*Model Answer:* A mature answer avoids both extremes (blind early adoption and reflexive skepticism). Look for a framework: start with a narrow, low-risk pilot with clear success metrics, measure against the *actual* bottleneck it claims to solve (not a vanity metric), and require the new approach to fit into existing governance/audit/security controls rather than being special-cased. This mirrors exactly how they'd want AI coding agents introduced per the JD.

---

## 10. Quick-Fire Technical Screening Questions
*(Use these as rapid depth-checks; a 16-year candidate should answer these fluently and move fast.)*

| # | Question | What a Strong Answer Covers |
|---|----------|------------------------------|
| 24 | What's the difference between `terraform`-style declarative infra and Ansible's largely imperative model, and where does Packer sit? | Packer is imperative *image-build time* automation; Ansible can be both (playbooks are imperative execution of a desired-state goal); distinguishes build-time vs runtime automation. |
| 25 | What's a PodDisruptionBudget and why does it matter during a node repave? | Ensures a minimum number/percentage of pods stay available during voluntary disruptions like node drains — critical to zero-downtime repaves. |
| 26 | Name two ways to achieve idempotency in a Bash-based provisioning script that isn't natively idempotent like Ansible. | Check-before-act patterns (test state, only act if needed), use of `set -e` with guarded conditionals, or wrapping in tools like `flock`/marker files to prevent duplicate execution. |
| 27 | What's the difference between a Red Hat UBI image and a Chainguard image philosophically? | UBI = enterprise-supported, glibc-compatible, still has a package manager/shell; Chainguard = distroless-style, minimal attack surface, often no shell, built for supply-chain security (SBOMs, signed by default). |
| 28 | Why might `terraform`/pipeline state locking matter even in an image-build pipeline? | Prevents concurrent builds/promotions from corrupting shared state (e.g., an image registry manifest or a lineage-tracking DB) — same principle as Terraform state locking discussed generally. |
| 29 | What does "increasingly accelerated by AI coding agents" imply for how you write onboarding documentation and playbooks now? | Docs/playbooks need to be structured and explicit enough for an agent to consume and act on reliably — this changes documentation standards, not just human onboarding. |

---

### Suggested Interview Structure (60–75 min)
1. **5 min** — Walk me through your platform engineering journey and biggest scale challenge (context-setting).
2. **20 min** — Deep dive on 2–3 questions from Sections 1–4 (core platform/lifecycle depth).
3. **15 min** — Sections 5–6 (full-stack tooling + AI agent judgment) — this differentiates candidates who are current vs. dated.
4. **10 min** — One security question (Section 7).
5. **15 min** — One SRE/incident story (Section 8) + one leadership/behavioral (Section 9), STAR format.
6. **5–10 min** — Quick-fire round (Section 10) to check breadth and fluency under time pressure.

**Red flags for a claimed 16-year candidate:**
- Can describe tools but not trade-offs or failure modes.
- No concrete incident/war stories, or stories with no systemic follow-up fix.
- Treats AI coding agents as either magic or irrelevant, with no governance thinking.
- Can't speak to the organizational/change-management side of driving fleet-wide automation adoption — purely technical framing at this seniority is itself a gap.