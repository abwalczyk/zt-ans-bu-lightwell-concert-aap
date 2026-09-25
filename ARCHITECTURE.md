# Concert + AAP + Lightwell Architecture

## Honest Value Proposition

### Concert's UNIQUE Value
1. **Pre-Indexed SBOM Inventory Database**
   - Instant portfolio-wide queries: "Which of our 500 apps still run pyyaml < 6.0.2?"
   - Answer in milliseconds (vs. scanning 500 repos at query time)
   
2. **Cross-Application Dependency Topology (Arena View)**
   - Enterprise-wide graph: Repository → Build Artifact → Deployments → Dependent Apps
   - Shows impact across team boundaries: "Patching this breaks 3 apps owned by other teams"

### AO's Role (Orchestration + Decision Logic)
- Receives Lightwell package events via EDA
- Queries Concert for SBOM/topology data
- Makes routing decisions (based on environment, severity, business rules)
- Orchestrates the update workflow
- Provides audit trail

### EDA's Role (Always-On Monitoring)
- Receives Artifactory webhooks when Lightwell publishes a remediated package
- Triggers AO workflows
- No polling of CVE feeds or artifact repositories required

### Tekton's Role (Continuous Integration)
- Builds and tests the updated application
- Produces a signed container image
- Updates the GitOps manifest with the new image tag

### ArgoCD's Role (Continuous Delivery)
- Watches the GitOps repository for manifest changes
- Auto-syncs desired state to OpenShift
- Provides drift detection, health status, and one-command rollback

---

## Simplified Flow

```
Lightwell publishes pyyaml 6.0.2 to Artifactory
        ↓
┌───────────────────┐
│   EDA Webhook     │ → Receives package event, triggers AO
└───────────────────┘
        ↓
┌───────────────────┐
│ Automation        │ → Primary orchestrator
│ Orchestrator      │
└───────────────────┘
        ↓
┌───────────────────┐
│ Concert Query 1:  │ → "Which apps run an older pyyaml?"
│ SBOM Inventory    │    Response: 12 apps (flask-api-app included)
└───────────────────┘
        ↓
┌───────────────────┐
│ Concert Query 2:  │ → "What's flask-api-app's topology?"
│ Arena View        │    Response: Prod deployment, 1 downstream dependency
└───────────────────┘
        ↓
┌───────────────────┐
│ AO Switch Node    │ → Decision: Prod + critical = Auto-update
│ (AO's Logic)      │    (NOT Concert's decision)
└───────────────────┘
        ↓
┌───────────────────┐
│ Update Jobs       │ → Git commit → Tekton build → ArgoCD sync → Validate
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
| **EDA** | Artifactory package event ingestion, workflow triggering | Orchestration, decision logic |
| **Lightwell** | Remediated dependency artifacts | CVE detection, SBOM analysis |
| **Artifactory** | Package storage, publish notifications | Impact analysis, orchestration |
| **Tekton** | Build, test, image push, GitOps manifest update | Deployment, decision logic |
| **ArgoCD** | GitOps deployment, drift detection, rollback | Building images, dependency updates |

---

## Concert API Calls (Honest)

### 1. SBOM Inventory Query (by package)
```http
GET /api/v1/inventory/packages/pyyaml/applications
{ "older_than": "6.0.2" }
```
**Returns:**
```json
{
  "package_name": "pyyaml",
  "total_affected_applications": 12,
  "total_applications": 537,
  "affected_applications": [
    {
      "name": "flask-api-app",
      "version": "v1.0.0",
      "package": "pyyaml",
      "current_version": "5.4.1",
      "repository": "https://github.com/labuser/flask-api-app",
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
  "update_timestamp": "2026-09-25T14:23:45Z",
  "validation_passed": true,
  "deployment_method": "tekton-argocd",
  "container_image": "quay.io/labuser/flask-api-app:sha-abc123"
}
```

**Result:** Concert's inventory updated, future portfolio queries reflect new version

---

## AO Routing Logic (NOT Concert's)

AO Switch Node evaluates:
```python
if concert_has_prod_deployment and severity in ("critical", "high"):
    route = "auto-update"
elif concert_downstream_dependencies > 0:
    route = "create-approval-request"
else:
    route = "queue-for-scheduled-update"
```

**Why AO, not Concert?**
- AO has access to app metadata (environment tags, business criticality)
- AO can factor in non-Concert data (Splunk alerts, OPA policies)
- Easier to customize routing logic without changing Concert

---

## Lab Teaching Moment

**Without Concert (small scale, 1-2 apps):**
- AO could query SBOMs from Git repos or artifact registries directly
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
   - EDA receives Artifactory package events (always-on monitoring)
   - Concert doesn't need automation rules or workflows

3. **Proactive, Not Reactive**
   - The workflow starts when the *fix* is published, not when the *problem* is found
   - Lightwell artifact metadata arrives with the event, so no separate Lightwell API query is needed
   - One package publish can fan out to every affected application in the portfolio

4. **No BS About AI Risk Scoring**
   - Removed claims about "IBM AI prioritization"
   - AO handles routing logic using standard conditionals
   - Concert's value is pre-indexed SBOM data, not ML models

5. **GitOps as the Deployment Contract**
   - Tekton produces artifacts; ArgoCD owns what is running
   - Desired state lives in Git, so drift detection and rollback come free
   - AO validates against ArgoCD sync/health rather than SSH-ing into hosts

6. **Focus on Unique Capabilities**
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
