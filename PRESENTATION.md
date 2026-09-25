# Lab Intro Presentation — Proactive Dependency Remediation

**Concert + AAP + Lightwell + Tekton/ArgoCD**
Target length: ~10 minutes before hands-on.

---

## Slide 1 — The Problem: Patching Is Reactive and Slow

- A CVE drops. Now what?
  - Which of our applications actually use the affected package?
  - Who owns them? What breaks downstream if we patch?
  - Someone opens a spreadsheet. Weeks pass.
- The industry average for remediating a critical dependency CVE is measured in **weeks**, not minutes.
- The bottleneck is not the fix. The bottleneck is **knowing where the fix needs to go** and **getting it there safely**.

**Talk track:** Most vulnerability programs are a detection pipeline bolted onto a manual remediation process. We're going to invert that.

---

## Slide 2 — The Inversion: Start From the Fix, Not the Flaw

| Reactive (typical) | Proactive (this lab) |
|---|---|
| Trigger: scanner finds a CVE | Trigger: Lightwell publishes a remediated package |
| "We have a problem" | "We have a solution — go apply it" |
| Hunt for affected apps | Concert already knows |
| Manual PR, manual deploy | Tekton builds, ArgoCD deploys |

- Red Hat **Lightwell** publishes a hardened, remediated build of a package to Artifactory.
- That publish event *is* the trigger. The fix arrives before anyone files a ticket.

**Talk track:** This is the key idea of the lab. Everything downstream follows from moving the trigger.

---

## Slide 3 — The Cast: Who Does What

| Component | Role in this lab |
|---|---|
| **Artifactory** | Package store — emits the webhook when a Lightwell package lands |
| **Event-Driven Ansible** | Always-on listener — turns the event into a workflow |
| **IBM Concert** | Pre-indexed SBOM inventory + Arena View dependency topology |
| **Automation Orchestrator** | The brain — routing decisions, approval gates, audit trail |
| **AAP** | The hands — playbook execution |
| **Tekton** | CI — build, test, image, GitOps manifest bump |
| **ArgoCD** | CD — GitOps sync to OpenShift, drift detection, rollback |

**Talk track:** Note what Concert is *not* doing. It is not orchestrating. It's a very fast answer to "who is affected?" AO owns the decisions.

---

## Slide 4 — Why Concert Earns Its Place

Two things Concert does that AO genuinely cannot do cheaply:

1. **Pre-indexed SBOM inventory**
   - "Which of our 537 applications run pyyaml below 6.0.2?" → answered in milliseconds
   - The alternative is scanning 500+ repositories at query time
2. **Arena View cross-application topology**
   - Repository → build artifact → deployments → dependent applications
   - Shows impact *across team boundaries*: "patching this affects an app owned by the network team"

**Honest framing:** At 1–2 applications, Concert is overkill — AO could read SBOMs directly. At 500 applications across 50 teams, it's the difference between minutes and weeks. We demo with one app; imagine the portfolio.

---

## Slide 5 — The Flow (End to End)

```
Lightwell publishes pyyaml 6.0.2 → Artifactory
        ↓  webhook
Event-Driven Ansible
        ↓  run_workflow_template
Automation Orchestrator
        ├─ Concert: which apps run an older pyyaml?   → 12 of 537
        ├─ Concert: flask-api-app topology?           → prod, 3 replicas, 1 downstream dep
        └─ Switch: prod + critical → auto-update
        ↓
AAP: update requirements.txt, branch, commit, push
        ↓
Tekton PipelineRun: test → build image → push to Quay → bump GitOps manifest
        ↓
ArgoCD: auto-sync → rolling update → 3/3 pods healthy
        ↓
AAP: validate (sync status, pod readiness, /health reports 6.0.2)
        ↓
Concert: PATCH SBOM — portfolio drift 12 → 11
```

**~3 minutes, package publish to validated production deployment.**

---

## Slide 6 — What You'll Do (3 Modules)

**Module 1 — DETECT** (~15 min)
Publish the Lightwell package, watch EDA fire, read Concert's portfolio impact analysis and Arena View topology, follow the AO routing decision.

**Module 2 — ORCHESTRATE** (~25 min)
Watch the AO workflow drive the git commit, the Tekton PipelineRun, and the ArgoCD sync. See the new pods come up on the remediated package.

**Module 3 — VALIDATE** (~15 min)
Walk the full audit trail — Artifactory → EDA → AO → Concert → Tekton → ArgoCD → running pod — and confirm Concert's inventory reflects reality.

---

## Slide 7 — What to Watch For

- **The switch node.** That's where your policy lives. Change one condition and the whole risk posture changes.
- **The Concert query latency.** One API call replaces a portfolio scan. That's the entire value proposition in one HTTP request.
- **The ArgoCD sync.** Nobody SSH'd anywhere. Desired state went into Git; the cluster converged. Rollback is one command.
- **The audit trail.** Every hop is independently queryable. This is what an auditor actually wants.

---

## Slide 8 — Demo Questions / Discussion

Open questions — we want your opinions on where this should go next:

1. **Should we wrap everything in a ServiceNow ticket?**
   Auto-create a change record at the switch node, attach the Concert impact analysis and Tekton logs, close it on validation? Does that add governance or just add latency?

2. **How do we make this more OpenShift-native — or give it its own pizzazz?**
   Should more of this run as OpenShift-native primitives? What would make the demo *memorable* rather than just correct?

3. **Policy gates.**
   Where do OPA / Kyverno / ACS admission policies belong in this flow? Block the image at admission, or gate earlier in the pipeline? Who owns the policy?

4. **Red Hat Dependency Analytics.**
   Does RHDA overlap with Concert here, or complement it? Shift-left in the IDE vs. portfolio-wide inventory — is there a story where both run?

5. **Trusted Profile Analyzer.**
   Where does TPA fit for SBOM attestation and VEX? Should the Tekton pipeline emit a signed SBOM and push it to TPA, so the audit trail includes provenance, not just a version bump?

6. **And the one we keep arguing about:** should auto-update to production ever be fully hands-off, or is an approval gate always required — and if so, approval by whom?
