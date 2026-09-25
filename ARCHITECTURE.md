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

## How the Lab Binds to Automation Orchestrator

AO is **not part of the lab topology**. `config/instances.yaml` stands up a
Gitea container, a `control` VM and `rhel01` — there is no AO node. AO is a
shared instance that many labs register against, so each student's environment
has to introduce itself to AO at provision time, over the API.

That is what `lab/setup/configure-ao.yml` does, as Phase 2b of
`lab/setup/main.yml` (after AAP, because it resolves job templates by name):

1. Exchange the lab service account's `client_credentials` for a bearer token
   (`POST /api/v1/auth/token`, form-encoded — it is the only supported grant).
2. Create an AO credential holding this lab's AAP admin login, then an
   `ansible_automation_platform` integration pointing at the lab's **public**
   control route.
3. Poll the integration until `validation_status == "valid"` — this is the step
   that proves AO can actually reach the student's AAP.
4. `GET /api/v1/aap/job_templates?integration_id=…` and map the six job template
   names to whatever IDs this particular AAP assigned.
5. Render `lab/ao/lightwell-proactive-update.json` with those IDs, this lab's
   integration and credential, and a lab-scoped webhook path.
6. `POST /api/v1/workflows`, then publish the resulting version — a draft
   workflow's trigger is not live.
7. Stamp the webhook path onto the "AO - Trigger Proactive Update Workflow" job
   template so EDA's relay knows where to send events.

**Required environment variables.** `config/secrets.yaml` is tracked in git, so
nothing here may go in it. These follow the same `lookup('env', ...)` pattern as
`OCP_TOKEN` and `ARTIFACTORY_TOKEN`:

| Variable | Required? | What it is |
|---|---|---|
| `AO_CLIENT_ID` | **yes** | Service account client ID |
| `AO_CLIENT_SECRET` | **yes** | Service account client secret |
| `AO_URL` | no | AO base URL; defaults to the shared instance |
| `AAP_PUBLIC_URL` | no | Override for the control route; see below |
| `LAB_ID` | no | AO object name suffix; defaults to `GUID` |

Only the two service account values actually need setting. The public AAP URL is
derived from `GUID` and `DOMAIN`, which provisioning already has, as
`https://control-${guid}.${domain}` — the exact URL the Showroom **AAP** tab
points at (`ui-config.yml`). It is a public RHDP route, so the AO host can reach
it; other labs on this AO instance are registered the same way.

If `AO_CLIENT_ID` is unset, `configure-ao.yml` **skips itself** and the lab build
still succeeds — you get the AAP fallback workflow but no AO orchestration.

`AO_CLIENT_ID` / `AO_CLIENT_SECRET` belong to **one service account created by
hand, once**, on the AO instance — a lab cannot bootstrap its own AO identity.
That same service account is what the workflow's `eda_trigger` authorizes via
`authorized_service_account_ids`, so it is both the provisioner and the caller.

**Multi-student caveat.** Every lab leaves a workflow, an integration and a
credential on the shared AO instance, all named for `lab_id`. Nothing reaps
them, and a stale integration shows `validation_status: "error"` in the AO
console once its AAP disappears. Run `lab/setup/teardown-ao.yml` on teardown, or
by hand against an old `LAB_ID`.

---

## Demo Mode

`lab_demo_mode` (env `CONCERT_DEMO_MODE`, **default `true`**) makes the six AAP
playbooks report representative results instead of calling a backing service.
It exists because two dependencies are not in place yet:

- The shared Concert instance is loaded with `juice-shop` data, so a query for
  `pyyaml` in `flask-api-app` legitimately returns nothing.
- `config/instances.yaml` provisions no OpenShift cluster, so the Tekton and
  ArgoCD nodes have nothing to talk to.

Each playbook emits the **same `set_stats` keys** either way, so the AO workflow
is unchanged and runs end to end with every node reporting plausible data. Each
demo-mode playbook also prints a `DEMO MODE:` line naming exactly what it did
not do, so the job output never misrepresents itself.

Set `CONCERT_DEMO_MODE=false` once this lab's SBOM is in Concert and the OCP
cluster exists. Nothing else changes.

`demo_downstream` (extra_var on **Concert - Query Topology**, default `0`)
controls the routing branch: `0` downstream consumers routes to `auto_update`
and runs straight through to the audit summary; `1` or more routes through the
human approval gate.

### Concert API contract

Read off the **API key** dialog in the Concert console:

```
base:    https://<host>:12443/concert/core/api/v2
headers: Authorization: C_API_KEY <key>
         InstanceId: <instance id>
```

The scheme is `C_API_KEY`, **not** `Bearer`, and `InstanceId` is required. The
key is delivered to the execution environments by the **IBM Concert** custom
credential type (`configure-aap.yml`), which injects `CONCERT_API_URL`,
`CONCERT_INSTANCE_ID` and `CONCERT_API_TOKEN` — an EE cannot read the control
VM's environment, so a plain `lookup('env', ...)` would always be empty.

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
