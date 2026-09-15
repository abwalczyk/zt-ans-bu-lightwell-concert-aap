# Concert + AAP + Lightwell CVE Remediation Lab - Workflow

## Complete Lab Flow (Numbered Steps)

### Phase 1: CVE Discovery (Concert)

**1. Concert Ingests SBOM**
   - SBOM uploaded to Concert (manually or via CI/CD)
   - Contains: flask-api-app with pyyaml 5.4.1

**2. Concert Detects CVE**
   - Concert correlates SBOM with vulnerability database
   - Finds: CVE-2026-52891 (PyYAML Deserialization)
   - CVSS: 9.8 (Critical)
   - CWE: CWE-502 (Deserialization of Untrusted Data)

**3. Concert Performs Triage**
   - Assesses severity: Critical
   - Checks business context: Production application
   - IBM Risk Score: 95
   - Remediation available: YES (Lightwell)

**4. Concert Automation Rule Triggers**
   - Rule condition met: CVSS >= 7.0
   - Action: Trigger Concert Workflow

**5. Concert Workflow Executes**
   - Block 1: Extract CVE context from Concert event
   - Block 2: Determine lab_id from application name
   - Block 3: HTTP Request → AAP API
   - Block 4: Log success

---

### Phase 2: Remediation Orchestration (AAP)

**6. AAP Receives Concert Payload**
   - Workflow: "Concert CVE Remediation Orchestrator"
   - Extra vars received:
     ```json
     {
       "cve_id": "CVE-2026-52891",
       "severity": "critical",
       "affected_dependency": "pyyaml",
       "lab_id": "lab-27"
     }
     ```

**7. AAP Job 1: Fetch Lightwell Artifact**
   - Playbook: `lab/playbooks/aap/fetch-lightwell-artifact.yml`
   - Query Lightwell API for CVE-2026-52891 remediation
   - Response: pyyaml 6.0.2 (fixed version)
   - Output: artifact URL, checksum, fixed_version

**8. AAP Job 2: Update Dependency in Gitea**
   - Playbook: `lab/playbooks/aap/update-dependency-gitea.yml`
   - Clone Gitea repository: labuser/flask-api-app
   - Create branch: `cve-remediation/CVE-2026-52891`
   - Update requirements.txt:
     ```
     pyyaml==5.4.1  →  pyyaml==6.0.2
     ```
   - Commit message: "Remediate CVE-2026-52891 via Lightwell"
   - Push to Gitea

**9. Gitea CI/CD Pipeline Auto-Triggers**
   - Gitea Actions detects branch push
   - Workflow: `.gitea/workflows/build.yml`
   - Steps:
     - Checkout code
     - Set up Python 3.11
     - Install dependencies (pip install -r requirements.txt)
     - Run tests (pytest)
     - Deploy to rhel01 (/opt/flask-app/)
     - Restart flask-app.service

**10. Gitea Pipeline Completion**
   - Build: SUCCESS
   - Tests: PASSED
   - Deployment: COMPLETE
   - Application restarted with PyYAML 6.0.2

**11. AAP Job 3: Validate Remediation**
   - Playbook: `lab/playbooks/validation/validate-remediation.yml`
   - Check deployed PyYAML version: `pip show pyyaml`
   - Expected: 6.0.2 ✓
   - Test application health: `curl http://localhost:5000/health`
   - Response: `{"status":"healthy","pyyaml_version":"6.0.2"}` ✓
   - Test YAML parsing endpoint
   - Validation: SUCCESS

**12. AAP Reports Results**
   - Workflow status: SUCCESSFUL
   - Set stats for audit trail:
     ```yaml
     validation_status: success
     deployed_version: 6.0.2
     cve_resolved: true
     validation_timestamp: 2026-09-15T14:23:45Z
     ```

---

### Phase 3: Audit Trail Review

**13. Concert Updates CVE Status** (optional)
   - AAP POSTs remediation status back to Concert API
   - Concert marks CVE-2026-52891 as RESOLVED for flask-api-app

**14. Complete Audit Trail Available**
   - Concert: CVE discovery, triage decision, workflow execution
   - AAP: Job execution history, workflow nodes, timestamps
   - Gitea: Git commit, pipeline logs, deployment history
   - Application: Deployed version, health checks

---

## Arrow Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    PHASE 1: CVE DISCOVERY                       │
└─────────────────────────────────────────────────────────────────┘

  SBOM Upload
      ↓
  ┌────────────────┐
  │    Concert     │ → Detects CVE-2026-52891 in pyyaml 5.4.1
  │ Vulnerability  │ → CVSS 9.8 (Critical)
  │   Detection    │ → Lightwell remediation available
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │    Concert     │ → IBM Risk Score: 95
  │     Triage     │ → Priority: URGENT
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │    Concert     │ → Condition: CVSS >= 7.0 ✓
  │ Automation Rule│ → Action: Trigger Workflow
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │    Concert     │ → Extract CVE context
  │    Workflow    │ → Determine lab_id
  │   (4 blocks)   │ → HTTP Request to AAP API
  └────────┬───────┘
           │
           │ POST https://control/api/v2/workflow_job_templates/42/launch/
           │ Body: {"extra_vars": {"cve_id": "CVE-2026-52891", ...}}
           ↓

┌─────────────────────────────────────────────────────────────────┐
│              PHASE 2: REMEDIATION ORCHESTRATION                 │
└─────────────────────────────────────────────────────────────────┘

  ┌────────────────┐
  │  AAP Workflow  │ → Receives Concert payload
  │   Triggered    │ → Workflow: Concert CVE Remediation Orchestrator
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │   AAP Job 1    │ → Query Lightwell API
  │ Fetch Lightwell│ → GET /api/v1/remediation?cve=CVE-2026-52891
  │    Artifact    │ → Response: pyyaml 6.0.2 available
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │   AAP Job 2    │ → Clone Gitea repo
  │Update Gitea    │ → Create branch: cve-remediation/CVE-2026-52891
  │   Dependency   │ → Update requirements.txt: pyyaml==6.0.2
  │                │ → Commit & push to Gitea
  └────────┬───────┘
           │
           │ git push origin cve-remediation/CVE-2026-52891
           ↓
  ┌────────────────┐
  │     Gitea      │ → Webhook triggers Gitea Actions
  │   CI/CD Pipeline│
  └────────┬───────┘
           ↓
  ┌─────────────────────────────────────┐
  │  Gitea Actions Workflow             │
  ├─────────────────────────────────────┤
  │  1. Checkout code                   │
  │  2. Set up Python 3.11              │
  │  3. pip install -r requirements.txt │ ← PyYAML 6.0.2 installed
  │  4. pytest (run tests)              │
  │  5. Deploy to /opt/flask-app        │
  │  6. systemctl restart flask-app     │
  └────────┬────────────────────────────┘
           ↓
  ┌────────────────┐
  │     rhel01     │ → Flask app restarted
  │  Application   │ → Now running PyYAML 6.0.2
  │    Updated     │
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │   AAP Job 3    │ → pip show pyyaml → Version: 6.0.2 ✓
  │   Validate     │ → curl /health → pyyaml_version: 6.0.2 ✓
  │  Remediation   │ → curl /api/parse-yaml → Safe ✓
  └────────┬───────┘
           ↓
  ┌────────────────┐
  │  AAP Workflow  │ → Status: SUCCESS
  │   Complete     │ → All jobs passed
  └────────┬───────┘
           │
           │ POST to Concert API (optional)
           ↓

┌─────────────────────────────────────────────────────────────────┐
│                 PHASE 3: AUDIT TRAIL REVIEW                     │
└─────────────────────────────────────────────────────────────────┘

  ┌────────────────┐
  │    Concert     │ → CVE-2026-52891: RESOLVED
  │  CVE Status    │ → Remediation: Lightwell 6.0.2
  │    Updated     │ → Timestamp: 2026-09-15T14:23:45Z
  └────────────────┘

  ┌────────────────┐
  │  AAP Audit     │ → Workflow execution: 4 min 32 sec
  │     Trail      │ → Job logs, timestamps, approvers
  └────────────────┘

  ┌────────────────┐
  │  Gitea Audit   │ → Commit SHA: abc123
  │     Trail      │ → Pipeline logs, deployment history
  └────────────────┘

  ┌────────────────┐
  │  Application   │ → Deployed: pyyaml 6.0.2
  │    Verified    │ → Health: OK
  │                │ → Vulnerability: RESOLVED
  └────────────────┘
```

---

## Simplified 3-Step View

```
   DISCOVER              ORCHESTRATE             VALIDATE
   ─────────             ───────────             ────────

1. Concert        →    2. AAP Workflow    →    3. Validation
   ├─ Detects CVE         ├─ Fetch Lightwell      ├─ Check version
   ├─ Calculates risk     ├─ Update Gitea         ├─ Test app health
   └─ Triggers AAP        ├─ Wait for CI/CD       └─ Confirm CVE gone
                          └─ Deploy & validate
```

---

## Module Breakdown (Lab Guide Structure)

### Module 1: DISCOVER — Concert CVE Detection (Steps 1-5)
Students will:
- View Concert CVE dashboard
- Observe CVE-2026-52891 detected in flask-api-app
- Review Concert triage decision (CVSS 9.8, Lightwell available)
- Watch Concert Workflow trigger AAP

### Module 2: ORCHESTRATE — AAP Remediation (Steps 6-12)
Students will:
- Monitor AAP workflow execution
- Observe Lightwell artifact retrieval
- Watch Gitea repository update
- Monitor Gitea CI/CD pipeline
- Review AAP validation job

### Module 3: VALIDATE — Remediation Verification (Steps 13-14)
Students will:
- Verify PyYAML version upgraded to 6.0.2
- Test application health endpoint
- Review complete audit trail
- Generate compliance report

---

## Timeline (Typical Execution)

| Time | Step | Component |
|------|------|-----------|
| T+0s | CVE detected | Concert |
| T+5s | Automation rule triggers | Concert |
| T+10s | Workflow executes | Concert |
| T+15s | AAP workflow starts | AAP |
| T+30s | Lightwell query complete | AAP + Lightwell |
| T+45s | Gitea updated | AAP + Gitea |
| T+50s | CI/CD pipeline starts | Gitea Actions |
| T+2m | Pipeline completes | Gitea Actions |
| T+2m 15s | Application restarted | rhel01 |
| T+2m 30s | Validation complete | AAP |
| T+2m 45s | Concert updated | Concert |

**Total time: ~3 minutes from CVE detection to validated remediation**

---

## Data Flow

```
Concert Event Payload
        ↓
{
  "cve_id": "CVE-2026-52891",
  "severity": "critical",
  "cvss_score": 9.8,
  "affected_packages": ["pyyaml:5.4.1"],
  "affected_applications": ["flask-api-app-lab-27"]
}
        ↓
Concert Workflow Transforms
        ↓
{
  "cve_id": "CVE-2026-52891",
  "severity": "critical",
  "affected_dependency": "pyyaml",
  "lab_id": "lab-27"
}
        ↓
AAP Workflow Receives
        ↓
Job 1 Output: {fixed_version: "6.0.2", artifact_url: "..."}
        ↓
Job 2 Output: {gitea_branch: "cve-remediation/CVE-2026-52891"}
        ↓
Job 3 Output: {deployed_version: "6.0.2", cve_resolved: true}
```
