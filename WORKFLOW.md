# Concert + AAP + Lightwell CVE Remediation Lab - Workflow

## Architecture Overview

**Concert** = Intelligence/Data Source (SBOM correlation, risk assessment, priority)  
**Automation Orchestrator** = Primary Orchestrator (workflow execution, decision making)  
**AAP** = Execution Engine (playbook runs, Lightwell integration, Gitea updates)

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

**3. AO Task Agent: Query Concert for SBOM Correlation**
   - Agent prompt: "Query Concert API to find which applications are affected by CVE-2026-52891"
   - Concert API call: `GET /api/v1/vulnerabilities/CVE-2026-52891/affected-applications`
   - Concert response:
     ```json
     {
       "cve_id": "CVE-2026-52891",
       "affected_applications": [
         {
           "name": "flask-api-app",
           "package": "pyyaml",
           "current_version": "5.4.1",
           "fixed_version": "6.0.2"
         }
       ],
       "sbom_matched": true
     }
     ```

**4. AO Task Agent: Query Concert for Risk Assessment**
   - Agent prompt: "Get Concert risk assessment for CVE-2026-52891 affecting flask-api-app"
   - Concert API call: `GET /api/v1/risk-assessment?cve=CVE-2026-52891&app=flask-api-app`
   - Concert response:
     ```json
     {
       "cve_id": "CVE-2026-52891",
       "application": "flask-api-app",
       "ibm_risk_score": 95,
       "cvss_score": 9.8,
       "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
       "severity": "critical",
       "priority": "urgent",
       "exploitability": "high",
       "business_impact": "high",
       "remediation_available": true,
       "recommended_action": "auto-remediate"
     }
     ```

**5. AO Switch Node: Routing Decision**
   - Based on Concert's risk score and remediation availability
   - Switch conditions:
     - **risk_score >= 90 AND remediation_available** → Auto-remediate (our path)
     - **risk_score >= 90 AND NOT remediation_available** → Create ServiceNow incident
     - **risk_score < 90** → Queue for scheduled patching
   - Decision: **Auto-remediate** ✓

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

**11. AO Task Agent: Report to Concert (Optional)**
   - Agent prompt: "Update Concert to mark CVE-2026-52891 as RESOLVED for flask-api-app"
   - Concert API call: `PATCH /api/v1/vulnerabilities/CVE-2026-52891`
   - Body:
     ```json
     {
       "application": "flask-api-app",
       "status": "resolved",
       "remediation_method": "lightwell_artifact",
       "deployed_version": "6.0.2",
       "validation_passed": true,
       "remediation_timestamp": "2026-09-16T14:23:45Z",
       "ao_workflow_id": "workflow-123"
     }
     ```

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
  │  Task Agent 1          │ → Query Concert API
  │  "SBOM Correlation"    │    GET /api/v1/.../affected-apps
  └────────┬───────────────┘
           │
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (Intelligence Source)          │
  │  - SBOM correlation                                 │
  │  - Returns: affected apps, packages, versions       │
  └────────┬────────────────────────────────────────────┘
           │
           │ Response: { "affected_applications": ["flask-api-app"] }
           ↓
  ┌────────────────────────┐
  │  Task Agent 2          │ → Query Concert API
  │  "Risk Assessment"     │    GET /api/v1/risk-assessment
  └────────┬───────────────┘
           │
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (Intelligence Source)          │
  │  - Risk scoring (IBM: 95, CVSS: 9.8)               │
  │  - Priority determination (URGENT)                  │
  │  - Remediation availability (TRUE)                  │
  └────────┬────────────────────────────────────────────┘
           │
           │ Response: { "ibm_risk_score": 95, "priority": "urgent" }
           ↓
  ┌────────────────────────┐
  │   Switch Node          │
  │  Route Decision        │
  │  Critical + Fix → Auto │
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
  │  Task Agent 3          │ → PATCH Concert API
  │  Report to Concert     │    Update CVE status → RESOLVED
  └────────┬───────────────┘
           ↓
  ┌─────────────────────────────────────────────────────┐
  │              CONCERT (Status Update)                │
  │  - CVE-2026-52891: RESOLVED                         │
  │  - flask-api-app: REMEDIATED                        │
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
   └─ Concert risk       ├─ CI/CD deploy       └─ Report to Concert
                         └─ Validation
```

---

## Module Breakdown (Lab Guide Structure)

### Module 1: TRIAGE — CVE Detection & Concert Intelligence (Steps 1-5)
**Students will:**
- Trigger CVE alert via EDA webhook
- Observe EDA rulebook activation
- Watch AO workflow start
- Review Concert SBOM correlation results
- Review Concert risk assessment (IBM Risk Score: 95, Priority: URGENT)
- Observe AO switch node routing decision

### Module 2: ORCHESTRATE — AO Remediation Workflow (Steps 6-11)
**Students will:**
- Monitor AO workflow execution
- Observe Lightwell artifact retrieval
- Watch Gitea repository update (commit + push)
- Monitor Gitea CI/CD pipeline
- Review AAP validation job results
- Observe Concert status update

### Module 3: VALIDATE — Audit Trail Review (Steps 12-14)
**Students will:**
- Review AO execution history (workflow, agents, jobs)
- Review Concert CVE dashboard (status: RESOLVED)
- Examine complete audit trail (EDA → AO → Concert → Gitea → App)
- Generate compliance report

---

## Timeline (Typical Execution)

| Time | Step | Component |
|------|------|-----------|
| T+0s | CVE alert received | EDA |
| T+2s | EDA rulebook triggers AO | EDA |
| T+5s | AO workflow starts | AO |
| T+10s | Concert SBOM query | AO → Concert |
| T+12s | Concert risk assessment | AO → Concert |
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
| **Concert Role** | Orchestrates everything | Intelligence/data source only |
| **Concert Actions** | HTTP Request to AAP, workflow execution | API queries only (SBOM, risk) |
| **AAP Integration** | Concert calls AAP API | AO calls AAP job templates |
| **Decision Making** | Concert Workflow blocks | AO Switch Node |
| **Audit Trail** | Concert-centric | AO-centric (Concert is one data point) |

**Why This Is Better:**
- ✅ Concert focused on its strength: intelligence & correlation
- ✅ AO controls entire workflow (better audit trail, approval gates, error handling)
- ✅ Easier to add non-Concert data sources (Splunk, OPA, etc.)
- ✅ More flexible routing (multi-path decisions, human approvals)
- ✅ Concert doesn't need workflow configuration (just API access)
