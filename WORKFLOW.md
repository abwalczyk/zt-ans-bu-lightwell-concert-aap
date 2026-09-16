# Concert + AAP + Lightwell CVE Remediation Lab - Workflow

## Architecture Overview

**Concert** = SBOM Inventory + Dependency Topology (which apps are affected, what are the dependencies)  
**Automation Orchestrator** = Primary Orchestrator (workflow execution, decision making, risk assessment)  
**AAP** = Execution Engine (playbook runs, Lightwell integration, Gitea updates)  
**EDA** = Event Monitoring (CVE alert ingestion, workflow triggering)

---

## Complete Lab Flow (14 Numbered Steps)

### **Module 1: CVE Detection & Triage**

**1. EDA Receives CVE Alert**
   - Webhook/event from vulnerability scanner, SIEM, or Red Hat Security Data API
   - EDA rulebook: `lab/extensions/eda/cve-alert.yml`
   - Payload received:
     ```json
     {
       "cve_id": "CVE-2026-52891",
       "source": "security-scanner",
       "package": "pyyaml"
     }
     ```

**2. EDA Triggers Automation Orchestrator Workflow**
   - EDA action: `run_workflow_template`
   - Workflow: "CVE Remediation Orchestrator"
   - Extra vars: `{ "cve_id": "CVE-2026-52891", "source": "security-scanner" }`

**3. AO Job Template: Query Concert SBOM Inventory**
   - Playbook: `lab/playbooks/concert/query-sbom-inventory.yml`
   - Concert API call: `GET /api/v1/inventory/vulnerabilities/CVE-2026-52891`
   - Concert response (from pre-indexed SBOM database):
     ```json
     {
       "cve_id": "CVE-2026-52891",
       "total_affected_applications": 12,
       "affected_applications": [
         {
           "name": "flask-api-app",
           "version": "v1.0.0",
           "package": "pyyaml",
           "current_version": "5.4.1",
           "repository": "labuser/flask-api-app",
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
     - **Environment = production AND CVSS >= 9.0** → Auto-remediate (our path)
     - **Environment = staging** → Queue for scheduled patching
     - **Has downstream_dependencies** → Create approval request
   - Decision for flask-api-app: **Auto-remediate** ✓ (production deployment, CVSS 9.8)

---

### **Module 2: Remediation Orchestration**

**6. AO Job Template: Fetch Lightwell Artifact**
   - Playbook: `lab/playbooks/aap/fetch-lightwell-artifact.yml`
   - Query Lightwell API for CVE-2026-52891 remediation
   - Lightwell response:
     ```json
     {
       "package": "pyyaml",
       "fixed_version": "6.0.2",
       "artifact_id": "pyyaml-6.0.2.rhlw-00456",
       "artifact_url": "https://lightwell.redhat.com/artifacts/...",
       "checksum": "sha256:abc123..."
     }
     ```
   - Set workflow stats: `lightwell_artifact`, `fixed_version`

**7. AO Job Template: Update Gitea Repository**
   - Playbook: `lab/playbooks/aap/update-dependency-gitea.yml`
   - Clone repository: `labuser/flask-api-app`
   - Create branch: `cve-remediation/CVE-2026-52891`
   - Update `requirements.txt`:
     ```diff
     flask==2.3.2
     -pyyaml==5.4.1
     +pyyaml==6.0.2  # Remediated CVE-2026-52891 via Lightwell
     requests==2.31.0
     ```
   - Commit: "Remediate CVE-2026-52891 via Lightwell artifact"
   - Push to Gitea

**8. AO Script Node: Monitor Gitea CI/CD Pipeline**
   - Gitea Actions auto-triggers on branch push
   - Script polls Gitea API for pipeline status
   - Pipeline steps:
     - Checkout code
     - Set up Python 3.11
     - `pip install -r requirements.txt` (installs PyYAML 6.0.2)
     - `pytest` (run tests)
     - Deploy to `/opt/flask-app` on rhel01
     - `systemctl restart flask-app`
   - Wait for completion (timeout: 10 minutes)

**9. Gitea Pipeline Completes**
   - Status: SUCCESS
   - Deployed version: pyyaml 6.0.2
   - Application restarted

**10. AO Job Template: Validate Remediation**
   - Playbook: `lab/playbooks/validation/validate-remediation.yml`
   - Checks performed:
     - Verify PyYAML version: `pip show pyyaml` → Version: 6.0.2 ✓
     - Test application health: `curl /health` → pyyaml_version: 6.0.2 ✓
     - Test YAML parsing: `curl /api/parse-yaml` → Safe ✓
   - Validation result: **PASSED**

**11. AO Job Template: Update Concert Inventory**
   - Playbook: `lab/playbooks/concert/update-inventory.yml`
   - Concert API call: `PATCH /api/v1/inventory/applications/flask-api-app`
   - Body (update SBOM with new PyYAML version):
     ```json
     {
       "package": "pyyaml",
       "previous_version": "5.4.1",
       "new_version": "6.0.2",
       "remediation_timestamp": "2026-09-16T14:23:45Z",
       "validation_passed": true
     }
     ```
   - **Result**: Concert's SBOM inventory updated, CVE-2026-52891 no longer shows flask-api-app as affected

---

### **Module 3: Audit Trail Review**

**12. Automation Orchestrator Execution History**
   - Workflow execution ID: workflow-123
   - All task agent outputs
   - All job template results
   - Switch node decision path
   - Complete timeline with timestamps

**13. Concert CVE Dashboard**
   - CVE-2026-52891 status: RESOLVED
   - Affected applications: flask-api-app (REMEDIATED)
   - Remediation timestamp: 2026-09-16T14:23:45Z
   - Link to AO workflow execution

**14. Complete Audit Trail Available**
   - EDA: Webhook receipt, rulebook activation
   - AO: Workflow execution, agent outputs, job results
   - Concert: API queries, risk assessment, status updates
   - Gitea: Git commit, pipeline logs
   - Application: Deployed version, health checks

---

## Arrow Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    CVE ALERT INGESTION (EDA)                    │
└─────────────────────────────────────────────────────────────────┘
                            ↓
  Webhook: POST /cve-alert
  Body: { "cve_id": "CVE-2026-52891", "source": "scanner" }
                            ↓
  ┌────────────────────────┐
  │   EDA Rulebook         │ → Trigger AO Workflow
  │  cve-alert.yml         │
  └────────┬───────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│       AUTOMATION ORCHESTRATOR (Primary Orchestrator)            │
└─────────────────────────────────────────────────────────────────┘
           ↓
  ┌────────────────────────┐
  │  Job Template 1        │ → Query Concert Inventory
  │  "SBOM Query"          │    GET /api/v1/inventory/vulnerabilities/CVE-2026-52891
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
  ┌────────────────────────┐
  │   Switch Node          │
  │  AO Route Decision     │
  │  Prod + CVSS9 → Auto   │
  └────────┬───────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────┐
│               REMEDIATION PATH (AAP Jobs)                       │
└─────────────────────────────────────────────────────────────────┘
           ↓
  ┌────────────────────────┐
  │  Job Template 1        │ → Query Lightwell API
  │  Fetch Lightwell       │    GET /api/v1/remediation
  └────────┬───────────────┘
           │
           │ Response: { "fixed_version": "6.0.2" }
           ↓
  ┌────────────────────────┐
  │  Job Template 2        │ → Git clone
  │  Update Gitea          │ → Update requirements.txt
  │                        │ → Git commit + push
  └────────┬───────────────┘
           │
           │ git push origin cve-remediation/CVE-2026-52891
           ↓
  ┌────────────────────────┐
  │  Script Node           │ → Poll Gitea API
  │  Monitor CI/CD         │ → Wait for pipeline completion
  └────────┬───────────────┘
           │
           ↓
  ┌─────────────────────────────────────┐
  │  Gitea Actions Pipeline             │
  ├─────────────────────────────────────┤
  │  1. Checkout code                   │
  │  2. Set up Python 3.11              │
  │  3. pip install pyyaml==6.0.2       │
  │  4. pytest (run tests)              │
  │  5. Deploy to /opt/flask-app        │
  │  6. systemctl restart flask-app     │
  └────────┬────────────────────────────┘
           │
           │ Pipeline: SUCCESS
           ↓
  ┌────────────────────────┐
  │  Job Template 3        │ → pip show pyyaml → 6.0.2 ✓
  │  Validate Remediation  │ → curl /health → OK ✓
  └────────┬───────────────┘
           │
           │ Validation: PASSED
           ↓
  ┌────────────────────────┐
  │  Job Template 4        │ → PATCH Concert Inventory
  │  Update SBOM           │    Update pyyaml: 5.4.1 → 6.0.2
  └────────┬───────────────┘
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (Inventory Update)             │
  │  - flask-api-app SBOM updated                       │
  │  - CVE-2026-52891: 12 → 11 affected apps            │
  └─────────────────────────────────────────────────────┘
```

---

## Simplified 3-Step View

```
   TRIAGE              ORCHESTRATE           VALIDATE
   ──────              ───────────           ────────

1. EDA + Concert   →  2. AO Workflow   →   3. Validation
   ├─ EDA webhook        ├─ Lightwell          ├─ Check version
   ├─ Concert SBOM       ├─ Gitea update       ├─ Test app
   ├─ Concert topology   ├─ CI/CD deploy       └─ Update Concert SBOM
   └─ AO routing         └─ Validation
```

---

## Module Breakdown (Lab Guide Structure)

### Module 1: TRIAGE — CVE Detection & Concert Inventory (Steps 1-5)
**Students will:**
- Trigger CVE alert via EDA webhook
- Observe EDA rulebook activation
- Watch AO workflow start
- Review Concert SBOM inventory results (12 affected apps from entire portfolio)
- Review Concert Arena View topology (deployments, dependencies)
- Observe AO switch node routing decision (based on deployment environment + CVSS)

### Module 2: ORCHESTRATE — AO Remediation Workflow (Steps 6-11)
**Students will:**
- Monitor AO workflow execution
- Observe Lightwell artifact retrieval
- Watch Gitea repository update (commit + push)
- Monitor Gitea CI/CD pipeline
- Review AAP validation job results
- Observe Concert inventory SBOM update

### Module 3: VALIDATE — Audit Trail Review (Steps 12-14)
**Students will:**
- Review AO execution history (workflow, jobs)
- Review Concert inventory (CVE-2026-52891: 12 → 11 affected apps)
- Review Concert Arena View (flask-api-app topology updated)
- Examine complete audit trail (EDA → AO → Concert → Gitea → App)
- Generate compliance report

---

## Timeline (Typical Execution)

| Time | Step | Component |
|------|------|-----------|
| T+0s | CVE alert received | EDA |
| T+2s | EDA rulebook triggers AO | EDA |
| T+5s | AO workflow starts | AO |
| T+10s | Concert SBOM inventory query | AO → Concert |
| T+12s | Concert Arena View topology query | AO → Concert |
| T+15s | Switch routes to auto-remediate | AO |
| T+20s | Lightwell query complete | AO → Lightwell |
| T+35s | Gitea updated | AO → Gitea |
| T+40s | CI/CD pipeline starts | Gitea Actions |
| T+2m | Pipeline completes | Gitea Actions |
| T+2m 15s | Application restarted | rhel01 |
| T+2m 30s | Validation complete | AO → rhel01 |
| T+2m 45s | Concert updated | AO → Concert |

**Total time: ~3 minutes from CVE alert to validated remediation**

---

## Key Differences from Old Architecture

| Aspect | OLD (Concert as Orchestrator) | NEW (AO as Orchestrator) |
|--------|------------------------------|--------------------------|
| **Trigger** | Concert Automation Rule | EDA Webhook → EDA Rulebook |
| **Orchestrator** | Concert Workflow | Automation Orchestrator |
| **Concert Role** | Orchestrates everything | SBOM inventory + topology mapping only |
| **Concert Actions** | HTTP Request to AAP, workflow execution | API queries only (inventory, topology) |
| **AAP Integration** | Concert calls AAP API | AO calls AAP job templates |
| **Decision Making** | Concert Workflow blocks | AO Switch Node (using app metadata + CVSS) |
| **Audit Trail** | Concert-centric | AO-centric (Concert is one data source) |

**Why This Is Better:**
- ✅ Concert focused on its **unique** strength: pre-indexed SBOM inventory at enterprise scale
- ✅ Concert Arena View provides cross-application topology (what breaks if you patch this)
- ✅ AO controls entire workflow (decision logic, approval gates, error handling)
- ✅ Easier to add non-Concert data sources (Snyk, Trivy, SAST tools)
- ✅ More flexible routing (multi-path decisions based on environment, team ownership)
- ✅ Concert doesn't need workflow configuration (just maintains SBOM database)
