# Concert + AAP + Lightwell Proactive Dependency Update Lab - Workflow

## Architecture Overview

**Concert** = SBOM Inventory + Dependency Topology (which apps are affected, what are the dependencies)  
**Automation Orchestrator** = Primary Orchestrator (workflow execution, decision making, risk assessment)  
**AAP** = Execution Engine (playbook runs, Lightwell integration, repository updates)  
**EDA** = Event Monitoring (Artifactory package notifications, workflow triggering)  
**Tekton** = CI/CD Pipeline (build, test, package)  
**ArgoCD** = GitOps Deployment (automated application deployment)

---

## Complete Lab Flow (14 Numbered Steps)

### **Module 1: Package Detection & Impact Analysis**

**1. EDA Receives Lightwell Package Notification**
   - Webhook/event from Artifactory when new Lightwell package lands
   - EDA rulebook: `lab/extensions/eda/lightwell-package-alert.yml`
   - Payload received:
     ```json
     {
       "package_name": "pyyaml",
       "package_version": "6.0.2",
       "artifact_id": "pyyaml-6.0.2.rhlw-00456",
       "repository": "lightwell-remediated",
       "artifact_url": "https://artifactory.example.com/lightwell-remediated/pyyaml-6.0.2.rhlw-00456",
       "checksum": "sha256:abc123...",
       "remediation_info": {
         "fixes_cve": ["CVE-2026-52891"],
         "severity": "critical"
       }
     }
     ```

**2. EDA Triggers Automation Orchestrator Workflow**
   - EDA action: `run_workflow_template`
   - Workflow: "Lightwell Proactive Update Orchestrator"
   - Extra vars: `{ "package_name": "pyyaml", "package_version": "6.0.2", "artifact_id": "pyyaml-6.0.2.rhlw-00456" }`

**3. AO Job Template: Query Concert SBOM Inventory**
   - Playbook: `lab/playbooks/concert/query-sbom-by-package.yml`
   - Concert API call: `GET /api/v1/inventory/packages/pyyaml/5.4.1/applications`
   - Concert response (from pre-indexed SBOM database):
     ```json
     {
       "package_name": "pyyaml",
       "queried_version": "5.4.1",
       "newer_version_available": "6.0.2",
       "total_affected_applications": 12,
       "affected_applications": [
         {
           "name": "flask-api-app",
           "version": "v1.0.0",
           "package": "pyyaml",
           "current_version": "5.4.1",
           "repository": "https://github.com/labuser/flask-api-app",
           "owner_team": "platform-team"
         },
         { "name": "payment-service", "package": "pyyaml", "current_version": "5.4.1" },
         { "name": "other-app-1", "package": "pyyaml", "current_version": "5.4.1" }
       ]
     }
     ```
   - **Concert's Value**: Already has all application SBOMs indexed (instant query vs. scanning 500 repos)

**4. AO Job Template: Query Concert Arena View (Topology)**
   - Playbook: `lab/playbooks/concert/query-topology.yml`
   - Concert API call: `GET /api/v1/arena/applications/flask-api-app/topology`
   - Concert response (from Arena View):
     ```json
     {
       "application": "flask-api-app",
       "repository": "labuser/flask-api-app",
       "build_artifacts": [
         { "type": "container", "image": "flask-api-app:v1.0.0" }
       ],
       "deployments": [
         { "environment": "production", "namespace": "prod", "replicas": 3 },
         { "environment": "staging", "namespace": "staging", "replicas": 1 }
       ],
       "downstream_dependencies": [
         { "name": "api-gateway", "owner_team": "network-team" }
       ]
     }
     ```
   - **Concert's Value**: Cross-application topology mapping (shows what breaks if you patch this)

**5. AO Switch Node: Routing Decision**
   - **AO's logic** (not Concert's) based on Concert's inventory data + app metadata
   - Switch conditions:
     - **Environment = production AND Lightwell package = critical/high severity** → Auto-update (our path)
     - **Environment = staging** → Queue for scheduled update
     - **Has downstream_dependencies** → Create approval request
   - Decision for flask-api-app: **Auto-update** ✓ (production deployment, critical severity CVE fix)

---

### **Module 2: Update Orchestration**

**6. AO Job Template: Update Application Repository**
   - Playbook: `lab/playbooks/aap/update-dependency-git.yml`
   - Artifact metadata already available from EDA trigger (no additional Lightwell query needed)
   - Clone repository: `https://github.com/labuser/flask-api-app`
   - Create branch: `lightwell-update/pyyaml-6.0.2`
   - Update `requirements.txt`:
     ```diff
     flask==2.3.2
     -pyyaml==5.4.1
     +pyyaml==6.0.2  # Lightwell remediated package (fixes CVE-2026-52891)
     requests==2.31.0
     ```
   - Commit: "Update pyyaml to Lightwell remediated version 6.0.2"
   - Push to GitHub
   - Set workflow stats: `branch_name`, `commit_sha`

**7. AO Script Node: Monitor Tekton Pipeline**
   - Tekton EventListener auto-triggers PipelineRun on branch push
   - Script polls Tekton API for PipelineRun status
   - Pipeline steps (Tekton Tasks):
     - **git-clone**: Clone repository at commit SHA
     - **install-deps**: `pip install -r requirements.txt` (installs PyYAML 6.0.2 from Artifactory)
     - **run-tests**: `pytest` (unit + integration tests)
     - **build-image**: Build container image with updated dependencies
     - **push-image**: Push to Quay.io registry
     - **update-gitops**: Update ArgoCD manifests with new image tag
   - Wait for completion (timeout: 10 minutes)

**8. Tekton Pipeline Completes**
   - Status: SUCCESS
   - Container image: `quay.io/labuser/flask-api-app:sha-abc123`
   - ArgoCD manifest updated: `apps/flask-api-app/overlays/production/kustomization.yaml`

**9. ArgoCD Deploys Updated Application**
   - ArgoCD detects manifest change (GitOps repository)
   - Auto-sync enabled: Deployment initiated
   - ArgoCD applies updated Deployment with new image
   - Rolling update strategy: New pods with PyYAML 6.0.2
   - Health check: All pods ready
   - Deployment complete

**10. AO Job Template: Validate Deployment**
   - Playbook: `lab/playbooks/validation/validate-deployment.yml`
   - Checks performed:
     - Query ArgoCD sync status: `argocd app get flask-api-app` → Sync Status: Synced, Health: Healthy ✓
     - Verify running pods: `oc get pods -n production` → 3/3 Running ✓
     - Check container image: `oc get deployment flask-api-app -o jsonpath='{.spec.template.spec.containers[0].image}'` → quay.io/labuser/flask-api-app:sha-abc123 ✓
     - Test application health: `curl https://flask-api-app.apps.ocp.example.com/health` → pyyaml_version: 6.0.2 ✓
     - Test functionality: `curl https://flask-api-app.apps.ocp.example.com/api/parse-yaml` → Safe ✓
   - Validation result: **PASSED**

**11. AO Job Template: Update Concert Inventory**
   - Playbook: `lab/playbooks/concert/update-inventory.yml`
   - Concert API call: `PATCH /api/v1/inventory/applications/flask-api-app/sbom`
   - Body (update SBOM with new PyYAML version):
     ```json
     {
       "package": "pyyaml",
       "previous_version": "5.4.1",
       "new_version": "6.0.2",
       "update_timestamp": "2026-09-25T14:23:45Z",
       "validation_passed": true,
       "deployment_method": "tekton-argocd",
       "container_image": "quay.io/labuser/flask-api-app:sha-abc123"
     }
     ```
   - **Result**: Concert's SBOM inventory updated to reflect new package version

---

### **Module 3: Audit Trail Review**

**12. Automation Orchestrator Execution History**
   - Workflow execution ID: workflow-123
   - All task agent outputs
   - All job template results
   - Switch node decision path
   - Complete timeline with timestamps

**13. Concert Dependency Dashboard**
   - Package pyyaml 5.4.1 → 6.0.2 across portfolio
   - Affected applications: flask-api-app (UPDATED)
   - Update timestamp: 2026-09-25T14:23:45Z
   - Link to AO workflow execution

**14. Complete Audit Trail Available**
   - Artifactory: Package publish event, artifact metadata, checksum
   - EDA: Webhook receipt, rulebook activation
   - AO: Workflow execution, agent outputs, job results
   - Concert: API queries, impact analysis, SBOM updates
   - Git: Commit SHA, branch, diff
   - Tekton: PipelineRun logs, task results, image digest
   - ArgoCD: Sync history, revision, health status
   - Application: Running image, pod status, health checks

---

## Arrow Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│           LIGHTWELL PACKAGE PUBLISHED (Artifactory)             │
└─────────────────────────────────────────────────────────────────┘
                            ↓
  Artifactory webhook: POST /lightwell-package
  Body: { "package_name": "pyyaml", "package_version": "6.0.2", ... }
                            ↓
  ┌────────────────────────────┐
  │   EDA Rulebook             │ → Trigger AO Workflow
  │  lightwell-package-alert   │
  └────────┬───────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│       AUTOMATION ORCHESTRATOR (Primary Orchestrator)            │
└─────────────────────────────────────────────────────────────────┘
           ↓
  ┌────────────────────────┐
  │  Job Template 1        │ → Query Concert Inventory by package
  │  "SBOM Query"          │    GET /api/v1/inventory/packages/pyyaml/5.4.1/applications
  └────────┬───────────────┘
           │
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (SBOM Inventory)               │
  │  - Pre-indexed SBOM database                        │
  │  - Returns: 12 affected apps (from 500+ total)      │
  └────────┬────────────────────────────────────────────┘
           │
           │ Response: { "affected_applications": [{"name": "flask-api-app", ...}, ...] }
           ↓
  ┌────────────────────────┐
  │  Job Template 2        │ → Query Concert Arena View
  │  "Topology Query"      │    GET /api/v1/arena/applications/flask-api-app
  └────────┬───────────────┘
           │
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (Topology Mapping)             │
  │  - Shows deployments (prod, staging)                │
  │  - Shows downstream dependencies                    │
  └────────┬────────────────────────────────────────────┘
           │
           │ Response: { "deployments": [...], "downstream_dependencies": [...] }
           ↓
  ┌────────────────────────────┐
  │   Switch Node              │
  │  AO Route Decision         │
  │  Prod + critical → Auto    │
  └────────┬───────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│                  UPDATE PATH (AAP Jobs)                         │
└─────────────────────────────────────────────────────────────────┘
           ↓
  ┌────────────────────────┐
  │  Job Template 1        │ → Git clone
  │  Update App Repo       │ → Update requirements.txt
  │                        │ → Git commit + push
  └────────┬───────────────┘
           │
           │ git push origin lightwell-update/pyyaml-6.0.2
           ↓
  ┌────────────────────────┐
  │  Script Node           │ → Poll Tekton API
  │  Monitor PipelineRun   │ → Wait for pipeline completion
  └────────┬───────────────┘
           │
           ↓
  ┌─────────────────────────────────────┐
  │  Tekton PipelineRun                 │
  ├─────────────────────────────────────┤
  │  1. git-clone                       │
  │  2. install-deps (pyyaml 6.0.2)     │
  │  3. run-tests (pytest)              │
  │  4. build-image (buildah)           │
  │  5. push-image (quay.io)            │
  │  6. update-gitops (image tag)       │
  └────────┬────────────────────────────┘
           │
           │ Pipeline: SUCCESS → GitOps repo updated
           ↓
  ┌─────────────────────────────────────┐
  │  ArgoCD                             │
  ├─────────────────────────────────────┤
  │  - Detects manifest change          │
  │  - Auto-sync → Rolling update       │
  │  - Health check: 3/3 pods Ready     │
  └────────┬────────────────────────────┘
           │
           │ Sync: Synced / Healthy
           ↓
  ┌────────────────────────┐
  │  Job Template 2        │ → argocd app get → Healthy ✓
  │  Validate Deployment   │ → curl /health → 6.0.2 ✓
  └────────┬───────────────┘
           │
           │ Validation: PASSED
           ↓
  ┌────────────────────────┐
  │  Job Template 3        │ → PATCH Concert Inventory
  │  Update SBOM           │    Update pyyaml: 5.4.1 → 6.0.2
  └────────┬───────────────┘
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (Inventory Update)             │
  │  - flask-api-app SBOM updated                       │
  │  - Portfolio drift: 12 → 11 apps on old version     │
  └─────────────────────────────────────────────────────┘
```

---

## Simplified 3-Step View

```
   DETECT              ORCHESTRATE           VALIDATE
   ──────              ───────────           ────────

1. EDA + Concert   →  2. AO Workflow   →   3. Validation
   ├─ Artifactory        ├─ Git update         ├─ ArgoCD health
   ├─ Concert SBOM       ├─ Tekton build       ├─ Test app
   ├─ Concert topology   ├─ ArgoCD deploy      └─ Update Concert SBOM
   └─ AO routing         └─ Validation
```

---

## Module Breakdown (Lab Guide Structure)

### Module 1: DETECT — Package Publish & Concert Impact Analysis (Steps 1-5)
**Students will:**
- Publish a Lightwell package to Artifactory (or simulate the webhook)
- Observe EDA rulebook activation on the Artifactory event
- Watch AO workflow start
- Review Concert SBOM inventory results (12 affected apps from entire portfolio)
- Review Concert Arena View topology (deployments, dependencies)
- Observe AO switch node routing decision (based on deployment environment + severity)

### Module 2: ORCHESTRATE — AO Update Workflow (Steps 6-11)
**Students will:**
- Monitor AO workflow execution
- Watch application repository update (commit + push)
- Monitor Tekton PipelineRun (build, test, image push, GitOps update)
- Observe ArgoCD auto-sync and rolling deployment
- Review AAP validation job results
- Observe Concert inventory SBOM update

### Module 3: VALIDATE — Audit Trail Review (Steps 12-14)
**Students will:**
- Review AO execution history (workflow, jobs)
- Review Concert inventory (pyyaml 5.4.1: 12 → 11 affected apps)
- Review Concert Arena View (flask-api-app topology updated)
- Examine complete audit trail (Artifactory → EDA → AO → Concert → Tekton → ArgoCD → App)
- Generate compliance report

---

## Timeline (Typical Execution)

| Time | Step | Component |
|------|------|-----------|
| T+0s | Lightwell package published | Artifactory |
| T+2s | EDA rulebook triggers AO | EDA |
| T+5s | AO workflow starts | AO |
| T+10s | Concert SBOM inventory query | AO → Concert |
| T+12s | Concert Arena View topology query | AO → Concert |
| T+15s | Switch routes to auto-update | AO |
| T+30s | App repository updated | AO → Git |
| T+35s | Tekton PipelineRun starts | Tekton |
| T+2m | Pipeline completes, image pushed | Tekton → Quay |
| T+2m 10s | ArgoCD detects manifest change | ArgoCD |
| T+2m 30s | Rolling update complete, pods healthy | ArgoCD → OCP |
| T+2m 45s | Validation complete | AO → OCP |
| T+3m | Concert updated | AO → Concert |

**Total time: ~3 minutes from package publish to validated deployment**

---

## Key Differences from Old Architecture

| Aspect | OLD (CVE-triggered, Gitea) | NEW (Package-triggered, Tekton/ArgoCD) |
|--------|---------------------------|----------------------------------------|
| **Trigger** | CVE alert from scanner/SIEM | Lightwell package lands in Artifactory |
| **Posture** | Reactive (vulnerability found) | Proactive (fix is available) |
| **Lightwell Query** | AO queries Lightwell API mid-workflow | Artifact metadata arrives with the event |
| **Concert Query** | By CVE ID | By package name + version |
| **CI** | Gitea Actions | Tekton PipelineRun on OpenShift |
| **CD** | SSH deploy + systemctl restart | ArgoCD GitOps auto-sync |
| **Deploy Target** | rhel01 VM (`/opt/flask-app`) | OpenShift Deployment (3 replicas) |
| **Artifact** | Files on disk | Container image in Quay |
| **Rollback** | Manual re-deploy | `argocd app rollback` to prior revision |

**Why This Is Better:**
- ✅ Proactive: the workflow starts when the *fix* exists, not when the *problem* is found
- ✅ Concert focused on its **unique** strength: pre-indexed SBOM inventory at enterprise scale
- ✅ Concert Arena View provides cross-application topology (what breaks if you patch this)
- ✅ AO controls entire workflow (decision logic, approval gates, error handling)
- ✅ Cloud-native CI/CD: Tekton + ArgoCD are OpenShift-native, no external runners
- ✅ GitOps: desired state in Git, drift detection and rollback come free
- ✅ One Lightwell package publish can fan out to every affected application
