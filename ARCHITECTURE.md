# Concert + AAP + Lightwell Architecture

## Honest Value Proposition

### Concert's UNIQUE Value
1. **Pre-Indexed SBOM Inventory Database**
   - Instant portfolio-wide queries: "Which of our 500 apps are affected by CVE-2026-52891?"
   - Answer in milliseconds (vs. scanning 500 repos at query time)
   
2. **Cross-Application Dependency Topology (Arena View)**
   - Enterprise-wide graph: Repository → Build Artifact → Deployments → Dependent Apps
   - Shows impact across team boundaries: "Patching this breaks 3 apps owned by other teams"

### AO's Role (Orchestration + Decision Logic)
- Receives CVE alerts via EDA
- Queries Concert for SBOM/topology data
- Makes routing decisions (based on environment, CVSS, business rules)
- Orchestrates remediation workflow
- Provides audit trail

### EDA's Role (Always-On Monitoring)
- Receives CVE alerts from external sources
- Triggers AO workflows
- No need for Concert to poll CVE feeds

---

## Simplified Flow

```
CVE-2026-52891 Alert
        ↓
┌───────────────────┐
│   EDA Webhook     │ → Receives alert, triggers AO
└───────────────────┘
        ↓
┌───────────────────┐
│ Automation        │ → Primary orchestrator
│ Orchestrator      │
└───────────────────┘
        ↓
┌───────────────────┐
│ Concert Query 1:  │ → "Which apps are affected?"
│ SBOM Inventory    │    Response: 12 apps (flask-api-app included)
└───────────────────┘
        ↓
┌───────────────────┐
│ Concert Query 2:  │ → "What's flask-api-app's topology?"
│ Arena View        │    Response: Prod deployment, 1 downstream dependency
└───────────────────┘
        ↓
┌───────────────────┐
│ AO Switch Node    │ → Decision: Prod + CVSS 9.8 = Auto-remediate
│ (AO's Logic)      │    (NOT Concert's decision)
└───────────────────┘
        ↓
┌───────────────────┐
│ Remediation Jobs  │ → Lightwell → Gitea → CI/CD → Validate
└───────────────────┘
        ↓
┌───────────────────┐
│ Concert Update    │ → Update SBOM: pyyaml 5.4.1 → 6.0.2
└───────────────────┘
```

---

## What Each Component Does

| Component | Responsibility | Does NOT Do |
|-----------|---------------|-------------|
| **Concert** | SBOM inventory database, topology mapping | Workflow orchestration, risk prioritization, triggering AAP |
| **AO** | Workflow orchestration, routing decisions, audit trail | SBOM indexing, cross-app topology mapping |
| **EDA** | CVE alert ingestion, workflow triggering | Orchestration, decision logic |
| **Lightwell** | Remediated dependency artifacts | CVE detection, SBOM analysis |
| **Gitea** | Source code repository, CI/CD pipeline | Dependency remediation, validation |

---

## Concert API Calls (Honest)

### 1. SBOM Inventory Query
```http
GET /api/v1/inventory/vulnerabilities/CVE-2026-52891
```
**Returns:**
```json
{
  "total_affected_applications": 12,
  "total_applications": 537,
  "affected_applications": [
    {
      "name": "flask-api-app",
      "version": "v1.0.0",
      "package": "pyyaml",
      "current_version": "5.4.1",
      "repository": "labuser/flask-api-app",
      "owner_team": "platform-team"
    }
  ]
}
```

**Concert's Value:** Pre-indexed (instant query), enterprise-wide (537 apps scanned)

---

### 2. Arena View Topology Query
```http
GET /api/v1/arena/applications/flask-api-app/topology
```
**Returns:**
```json
{
  "application": "flask-api-app",
  "repository": "labuser/flask-api-app",
  "deployments": [
    { "environment": "production", "replicas": 3 },
    { "environment": "staging", "replicas": 1 }
  ],
  "downstream_dependencies": [
    { "name": "api-gateway", "owner_team": "network-team" }
  ]
}
```

**Concert's Value:** Cross-team topology (shows dependencies owned by other teams)

---

### 3. Inventory Update (Post-Remediation)
```http
PATCH /api/v1/inventory/applications/flask-api-app/sbom
```
**Body:**
```json
{
  "package": "pyyaml",
  "previous_version": "5.4.1",
  "new_version": "6.0.2",
  "remediation_timestamp": "2026-09-16T14:23:45Z",
  "validation_passed": true
}
```

**Result:** Concert's inventory updated, future CVE queries reflect new version

---

## AO Routing Logic (NOT Concert's)

AO Switch Node evaluates:
```python
if concert_has_prod_deployment and cvss_score >= 9.0:
    route = "auto-remediate"
elif concert_downstream_dependencies > 0:
    route = "create-approval-request"
else:
    route = "queue-for-scheduled-patching"
```

**Why AO, not Concert?**
- AO has access to app metadata (environment tags, business criticality)
- AO can factor in non-Concert data (Splunk alerts, OPA policies)
- Easier to customize routing logic without changing Concert

---

## Lab Teaching Moment

**Without Concert (small scale, 1-2 apps):**
- AO could query SBOMs from Gitea/artifact registries directly
- AO could analyze dependencies from Ansible facts
- Concert is overkill

**With Concert (enterprise scale, 500+ apps):**
- Instant portfolio-wide impact analysis (vs. scanning 500 repos)
- Cross-team dependency visibility (apps owned by different teams)
- Centralized SBOM database (single source of truth)

**Honest pitch:** "Concert shines at enterprise scale. In this lab, we're demonstrating the integration pattern with 1-2 apps, but imagine this with 500+ applications across 50 teams."

---

## Key Architectural Decisions

1. **Concert = Data, AO = Logic**
   - Concert provides SBOM inventory and topology data
   - AO makes all orchestration and routing decisions

2. **EDA Triggers, Not Concert**
   - EDA receives CVE alerts (always-on monitoring)
   - Concert doesn't need automation rules or workflows

3. **No BS About AI Risk Scoring**
   - Removed claims about "IBM AI prioritization"
   - AO handles routing logic using standard conditionals
   - Concert's value is pre-indexed SBOM data, not ML models

4. **Focus on Unique Capabilities**
   - Pre-indexed SBOM inventory (instant queries)
   - Cross-application topology mapping (Arena View)
   - These are things AO can't easily replicate

---

## For Partners/Customers

**When to use Concert:**
- You have 50+ applications across multiple teams
- You need instant "which apps are affected" answers
- You need cross-team dependency visibility

**When Concert is overkill:**
- You have 1-5 applications
- All apps owned by one team
- AO can query SBOMs directly from artifact registries

**This lab demonstrates:** The integration pattern at enterprise scale, using 1-2 apps for simplicity.
