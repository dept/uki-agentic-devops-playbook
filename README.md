# UKI Agentic DevOps Playbook

A practical guide for DevOps engineers getting started with AI-assisted infrastructure work. This playbook is based on real experience migrating the HSE (Health Service Executive) Ireland web platform from Azure Front Door to Cloudflare using BMAD and GitHub Copilot agents to plan and implement the work.

---

## When to use this playbook

**This playbook is for infrastructure that already exists.** BMAD works best when you have a live codebase, running infrastructure you can query, and a defined problem to solve — a migration, a cutover, a new module, a security hardening exercise. The HSE Cloudflare migration is a good example: existing Terraform repo, live Azure infrastructure queryable via CLI, a clear Phase 2 objective.

**For infrastructure that doesn't exist yet — use FORGE instead.**

If you are scoping a new platform, evaluating whether to build something, or exploring architecture options before any code or infrastructure exists, BMAD is the wrong starting point. FORGE is built for that phase — it pressure-tests the approach through structured interrogation before you commit to an architecture or start writing Terraform.

> A dedicated playbook for FORGE is planned: **[uki-forge-devops-playbook](https://github.com/dept/uki-forge-devops-playbook)** — not yet written, watch this space.

The rough decision:

| Situation | Tool |
|---|---|
| Existing infrastructure — migration, cutover, module changes | This playbook (BMAD) |
| Security hardening, compliance changes on live infrastructure | This playbook (BMAD) |
| New Terraform module for an existing platform | This playbook (BMAD) |
| New platform — nothing exists yet, architecture not decided | FORGE |
| Evaluating tools or approaches — "should we use Cloudflare Tunnels?" | FORGE |
| Proof of concept, spike, feasibility investigation | FORGE |
| Not sure | Start with FORGE, move to BMAD once the approach is decided |

---

It is not a tutorial on AI in general. It is a working guide for engineers who use Terraform, Terragrunt, Cloudflare, and Azure and want to understand how to bring AI agents into that workflow properly — with the right guardrails, context, and human gates.

---

## Contents

1. [Philosophy — why this works differently to vibe coding](#1-philosophy)
2. [Prerequisites](#2-prerequisites)
3. [Setting up BMAD in a repo](#3-setting-up-bmad)
4. [The full workflow — from problem to deployed infrastructure](#4-the-full-workflow)
5. [Verifying your infrastructure data before planning](#5-verifying-your-infrastructure-data)
6. [Writing architecture decisions for Terraform/Terragrunt](#6-writing-architecture-decisions)
7. [Writing GitHub issues an agent can act on](#7-writing-github-issues)
8. [Installing and using Terraform/Terragrunt agent skills](#8-terraform-skills)
9. [Human gates — what agents must never skip](#9-human-gates)
10. [Jira integration](#10-jira-integration)
11. [Lessons learned — real mistakes from the HSE migration](#11-lessons-learned)
12. [Quick reference](#12-quick-reference)

---

## 1. Philosophy

### Planning first, code second

The most common mistake with AI coding agents is jumping straight to implementation. You get plausible-looking Terraform that violates your module conventions, renames DNS records (causing a destroy/create instead of an in-place update), or misses a dependency chain entirely.

BMAD (Broad Method for Agentic Development) solves this by driving proper planning before any code is written:

1. **PRD** — what needs to happen and why, with explicit non-goals and open questions
2. **Architecture spine** — the invariants that govern implementation (which modules to use, what patterns are mandatory, what is explicitly prohibited)
3. **Epics and stories** — broken down with full acceptance criteria including exact resource names, expected HTTP responses, and testable conditions
4. **GitHub issues** — each story becomes a self-contained brief with everything the agent needs

By the time an agent picks up a story, it has:
- The project-context.md (repo-specific rules)
- The architecture spine (the decisions it must not violate)
- The infrastructure map (the exact FQDNs and routing rules)
- The acceptance criteria (what done looks like)

The agent writes the Terraform. You review the plan. You approve the apply. The human gates do not change.

### What changes

The time you spend shifts from writing repetitive Terraform to writing good requirements and reviewing plan outputs. The first is where mistakes happen silently. The second is where you actually want to spend your attention on infrastructure that serves a national health service.

---

## 2. Prerequisites

### Tools

```bash
# Core
brew install terraform terragrunt

# AI agent tooling
npm install -g @opencode/cli   # or use Claude Code / GitHub Copilot

# BMAD
# Installed per-repo via npx — no global install needed

# Terraform/Terragrunt skills
npx skills add https://github.com/hashicorp/agent-skills --skill terraform-style-guide --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terragrunt-generator --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terragrunt-validator --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terraform-validator --yes
```

### Accounts and access

- GitHub account with Copilot access (for agent assignment on issues)
- Jira API token if you want programmatic issue creation (generate at https://id.atlassian.com/manage-profile/security/api-tokens)
- Cloudflare API credentials if working on Cloudflare infrastructure
- Azure CLI authenticated if working on Azure infrastructure (`az login`)

---

## 3. Setting up BMAD

### In a new or existing repo

```bash
# Install BMAD into your repo
npx --yes tiged bmad-code-org/BMAD-METHOD/dist/packages/base _bmad
```

This creates `_bmad/` (the framework) and `.agents/skills/` (the skill library).

### Configure for your project

Edit `_bmad/config.toml`:

```toml
[core]
project_name = "your-project-name"
user_name = "Your Name"
communication_language = "English"
document_output_language = "English"
output_folder = "{project-root}/_bmad-output"

[modules.bmm]
planning_artifacts = "{project-root}/_bmad-output/planning-artifacts"
implementation_artifacts = "{project-root}/_bmad-output/implementation-artifacts"
project_knowledge = "{project-root}/docs"
user_skill_level = "intermediate"
```

### Install Terraform/Terragrunt skills

```bash
npx skills add https://github.com/hashicorp/agent-skills --skill terraform-style-guide --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terragrunt-generator --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terragrunt-validator --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terraform-validator --yes
```

### Register GitHub Copilot agents

Create `.github/agents/` directory with agent definition files. For a DevOps project you typically want at minimum:

```markdown
# .github/agents/bmad-agent-dev.agent.md
---
description: Senior software engineer for story execution and code implementation.
---

LOAD the FULL {project-root}/.agents/skills/bmad-agent-dev/SKILL.md, READ its entire contents and follow its directions exactly!
```

### Commit everything

```bash
git add .agents/ _bmad/ .github/agents/ skills-lock.json
git commit -m "chore: add BMAD agent framework and Terraform/Terragrunt skills"
```

> **Note on pre-commit hooks:** If your repo uses shellcheck, exclude `.agents/` — the external skill scripts may not pass shellcheck without changes you don't own:
>
> ```yaml
> # .pre-commit-config.yaml
> - id: shellcheck
>   exclude: ^\.agents/
> ```

---

## 4. The full workflow

```
Problem statement
      ↓
Generate project context  (bmad-generate-project-context)
      ↓
PRD                       (bmad-prd)
      ↓
Architecture spine        (bmad-architecture)
      ↓  ← human review gate
Epics and stories         (bmad-create-epics-and-stories)
      ↓
Sprint planning           (bmad-sprint-planning)
      ↓
GitHub issues created     (one per story)
      ↓
Assign issue to agent     ← you trigger this
      ↓
Agent writes Terraform / opens PR
      ↓
Review plan output        ← you review this
      ↓
NSG/CAB gate checks       ← human gates
      ↓
Approve apply             ← you approve this
      ↓
Smoke tests               ← you verify this
      ↓
Update Jira + sprint tracker
      ↓
Next story
```

### The generate project context step matters

Before any planning, run the `bmad-generate-project-context` skill. This is run **inside your AI agent tool** — OpenCode, Claude Code, or GitHub Copilot Chat — not from the terminal.

Open your agent tool in the repo, then:

```
/bmad-generate-project-context
```

Or in a chat window:

```
Run bmad-generate-project-context
```

The skill scans your existing repo and produces `_bmad-output/project-context.md` — a file the agent loads at runtime that tells it:

- Which modules exist and what they do
- Naming conventions
- What is explicitly prohibited
- Tool versions
- Critical patterns that prevent mistakes

For a Terraform repo this includes things like: "DNS edits are in-place only — never rename a map key", "blob paths go via cloud_connectors not origin_rules", "one cloudflare_ruleset per zone per phase". Without this file the agent guesses. With it, the agent knows.

---

## 5. Verifying your infrastructure data

### The lesson: never trust AI-generated infrastructure maps

When we started the HSE Front Door migration, we had an AI-generated CSV of routes and origins pulled from Front Door configuration. It was significantly wrong:

- Missing all `westeurope` (euwe) Container App replicas — only `northeurope` (euno) was listed
- Old decommissioned App Services still listed as active origins
- Only 3 Front Door profiles identified — there were actually 7 (separate profiles for vaccine and healthpromotion)
- Wrong CloudFront distribution IDs
- `dept.ie` scoped domains included that were out of scope

**Always pull from the live system before planning.** For Azure Front Door:

```bash
# List all Front Door profiles
az afd profile list --query "[].{name:name, rg:resourceGroup}" -o table

# For each profile, list origin groups
az afd origin-group list \
  --profile-name <profile> \
  --resource-group <rg> \
  --query "[].name" -o tsv

# For each origin group, list enabled origins
az afd origin list \
  --profile-name <profile> \
  --resource-group <rg> \
  --origin-group-name <group> \
  --query "[?enabledState=='Enabled'].{name:name, host:hostName}" -o table

# Get all routes
az afd route list \
  --profile-name <profile> \
  --resource-group <rg> \
  --endpoint-name <endpoint> \
  --query "[].{name:name, patterns:patternsToMatch, originGroup:originGroup.id}" -o json

# Get all custom domains
az afd custom-domain list \
  --profile-name <profile> \
  --resource-group <rg> \
  --query "[].{host:hostName, state:domainValidationState}" -o table
```

Do this for every profile. What we found when we did it properly for the HSE project:

- `wagtailMainBackendPool` had 4 origins — 2 Container Apps (euno + euwe) and 2 App Services — the AI map only showed 1
- `campaignNodejsMainBackendPool` was multi-region — the AI map missed this entirely
- 3 bare IP origins were discovered: `137.191.241.85`, `52.138.219.104`, `52.138.142.236` — none in the AI map
- 2 entirely unknown Front Door profiles for vaccine and health promotion

**These gaps would have caused production incidents if we had planned against the wrong data.**

### Generate corrected infrastructure maps

Once you have the live data, generate accurate CSVs. For the HSE project we used a format that captures what matters for a migration:

```
Scope,Front Door Profile,URL,Path,Container/Origin,Origin FQDN,Origin Type,Multi-Region,Notes
```

Store these in `docs/map/` and commit them. They become the source of truth for the architecture and every story's acceptance criteria.

---

## 6. Writing architecture decisions for Terraform/Terragrunt

The architecture spine is the most important document in the whole process. It contains Architecture Decisions (ADs) — the invariants that every story must respect. If an AD is wrong or missing, the agent will build the wrong thing.

### What makes a good AD for infrastructure

Each AD needs three things:
- **Binds:** what it applies to
- **Prevents:** the divergence it stops (two engineers building incompatibly)
- **Rule:** the enforceable constraint

**Example from the HSE migration:**

```markdown
### AD-5 [ADOPTED] DNS cutover mechanism

- **Binds:** all DNS record changes in domains/*/dns/
- **Prevents:** one engineer doing an in-place content edit while another renames the map key (which causes destroy/create)
- **Rule:** DNS cutover = in-place edit of existing DNS record content value only. Same Terragrunt map key, new content value. No resource recreation. Renaming the map key constitutes recreation. Add lifecycle { prevent_destroy = true } to all DNS records in cutover scope.
```

### ADs that matter most for Terraform/Terragrunt projects

1. **Module responsibility** — which module handles which resource type. Two implementers must not solve the same problem with different modules.
2. **Resource naming/keying** — especially for `for_each` maps. Key renames = destroy/create = potential outage.
3. **Lifecycle rules** — `prevent_destroy`, `ignore_changes` — when and where they are required.
4. **Provider scope** — what providers are generated by root config vs declared in modules. For Terragrunt projects using `generate "provider"` blocks, modules must never declare the provider themselves.
5. **State key format** — how state file paths are derived. One format, enforced.
6. **Dependency ordering** — which modules depend on which. The unit dependency graph is an invariant, not a suggestion.

### The reviewer pass matters

After drafting the spine, run two adversarial reviewer passes:

1. **Technology freshness** — are the named resource types still correct for the provider version pinned in the repo? Cloudflare provider v5 has different resource names than v4. Verify before locking ADs.

2. **Incompatibility check** — construct two implementers each following every AD. Can they still build incompatibly? If yes, tighten the AD. Common gaps:
   - AD says "use `host_header` for single-origin rules" but doesn't say what to use for path-splits → one engineer uses `host_header` everywhere, breaking path overrides
   - AD says "one ruleset per zone" but doesn't define what happens if a second feature needs the same phase → engineer creates a second ruleset, Terraform errors

---

## 7. Writing GitHub issues an agent can act on

A GitHub issue that produces good Terraform has these properties:

### User story with clear value

```
As a platform engineer,
I want to [specific Terraform change],
So that [measurable outcome].
```

### Exact origin data

Don't write "update the DNS record for www.hse.ie". Write:

```
Change the content value of the "www" CNAME record in 
domains/hse.ie/dns/terragrunt.hcl from 
fd-hse-prd-brd0gyc8escsd0as.z01.azurefd.net
to 
ca-hseie-main-euno-prd.whitehill-69b362a9.northeurope.azurecontainerapps.io
```

The agent cannot look up what the current value is unless you tell it. Exact values eliminate ambiguity and make the PR reviewable.

### Architectural constraints inline

Copy the relevant ADs from the architecture spine directly into the issue:

```
Key constraints:
- DNS edit is in-place only — same map key, new content value (AD-5)
- Add lifecycle { prevent_destroy = true } to this record
- Prepare rollback PR before apply — must be executable in <10 minutes
- NSG allowlist confirmation from Azure team required before apply (FR-12)
```

### Testable acceptance criteria

```
**Given** the DNS change is applied
**When** a DNS lookup is performed for www.hse.ie
**Then** the record resolves to a Cloudflare IP (not a Front Door IP)
**And** a request to https://www.hse.ie/ returns HTTP 200 with server: cloudflare
**And** no x-azure-fdid header is present in the response
```

These map directly to a smoke test the engineer runs after apply.

### Sprint tracker instruction

```
Update _bmad-output/implementation-artifacts/sprint-status.yaml:
- Set 2-3-update-dns-records-for-hse-ie-dev-hostnames: in-progress when starting
- Set 2-3-...: review when PR is open
- Set 2-3-...: done when merged
```

---

## 8. Terraform/Terragrunt agent skills

### The gap in BMAD

As of mid-2026, BMAD has no dedicated DevOps/infrastructure agent. The generic `bmad-agent-dev` skill is a senior software engineer — it knows how to implement stories but has no native Terraform/Terragrunt knowledge.

There is an open upstream issue for this: https://github.com/bmad-code-org/BMAD-METHOD/issues/2187

### The fix — external skills

Install these from the skills ecosystem. They give the agent real Terraform/Terragrunt knowledge:

```bash
# HashiCorp official style guide — 7.5k installs, security audited
npx skills add https://github.com/hashicorp/agent-skills --skill terraform-style-guide --yes

# Terragrunt scaffolding and validation
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terragrunt-generator --yes
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terragrunt-validator --yes

# Terraform validation (runs terraform fmt → tflint → validate → Checkov)
npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terraform-validator --yes
```

Commit the `skills-lock.json` this creates so the whole team uses pinned versions:

```bash
git add .agents/skills/ skills-lock.json
git commit -m "chore: add Terraform/Terragrunt agent skills"
```

### What these skills cover vs what project-context.md covers

| Source | Covers |
|---|---|
| `terraform-style-guide` | HCL formatting, file organisation, variable naming, provider patterns |
| `terragrunt-generator` | Root/child/stack layouts, `include` patterns, `dependency` blocks |
| `terragrunt-validator` | Validates HCL, module wiring, dependency chains |
| `terraform-validator` | Runs your actual toolchain (tflint, checkov, validate) |
| `project-context.md` | Repo-specific rules — your module interfaces, your naming conventions, your prohibited patterns |

The external skills handle general correctness. `project-context.md` handles the things that are unique to your repo. You need both.

---

## 9. Human gates

### What agents must never do without explicit approval

- Run `terraform apply` or `terragrunt apply`
- Modify production DNS records
- Delete any resource
- Change security group / NSG rules
- Rotate credentials or API tokens

These should be stated explicitly in your `.opencode/agents/default.md` or equivalent:

```markdown
NEVER run terragrunt apply or terraform apply without explicit user approval.
Always run plan first and present the output for review.
```

### Infrastructure-specific gates

Beyond the basic apply gate, infrastructure migrations have additional human gates that must be respected:

**NSG/firewall gate** — for any cutover that relies on Cloudflare proxying to an origin, the Azure team must confirm that NSG rules allowing Cloudflare IP ranges are in place before the DNS record is changed. An agent can write the Terraform. It cannot confirm the Azure-side gate. This must be explicitly documented in the story acceptance criteria and checked before apply.

**CAB gate** — for organisations with formal change management, PRD cutover stories require a CAB ticket reference before any apply. This is a process gate, not a technical one. The agent cannot raise a CAB ticket. The engineer does. The story should not be applied until the CAB reference is in the PR.

**Clinical hours gate** — for services that support clinical operations (A&E systems, emergency services), cutover must not happen during peak hours. This needs explicit timing agreed with operations before the cutover window opens.

**Rollback readiness gate** — before any cutover, a rollback Terraform change must be prepared, reviewed, and ready to merge. For DNS changes this means a PR that reverts the CNAME content back to the previous value, executable in under 10 minutes.

### The smoke test is not optional

Every cutover story should define a smoke test suite — a set of URLs, expected status codes, and expected/absent response headers. The agent can help write this based on the acceptance criteria. Running it is the engineer's job after apply.

For DNS cutovers the minimum smoke test is:
```bash
curl -sI https://your-domain.ie/ | grep -E "server|x-azure-fdid|HTTP"
```
You want to see `server: cloudflare` and no `x-azure-fdid`. If you see `x-azure-fdid` the traffic is still going through Front Door.

---

## 10. Jira integration

### Programmatic setup

Rather than manually creating 24 Jira stories, use the Jira REST API. You need:

- Your Atlassian email
- A Jira API token (https://id.atlassian.com/manage-profile/security/api-tokens)
- Your project key

```bash
export JIRA_API_TOKEN=your_token_here

# Check auth
curl -s -u "you@company.com:$JIRA_API_TOKEN" \
  "https://yourorg.atlassian.net/rest/api/3/myself" \
  -H "Accept: application/json" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('displayName'))"

# Get available issue types for your project
curl -s -u "you@company.com:$JIRA_API_TOKEN" \
  "https://yourorg.atlassian.net/rest/api/3/issue/createmeta/YOURPROJECT/issuetypes" \
  -H "Accept: application/json" | python3 -c "
import sys,json
for t in json.load(sys.stdin).get('issueTypes',[]):
    print(t['id'], '|', t['name'], '|', t.get('hierarchyLevel'))
"

# Get required fields for Stories
curl -s -u "you@company.com:$JIRA_API_TOKEN" \
  "https://yourorg.atlassian.net/rest/api/3/issue/createmeta?projectKeys=YOURPROJECT&issuetypeIds=STORY_TYPE_ID&expand=projects.issuetypes.fields" \
  -H "Accept: application/json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
fields=d['projects'][0]['issuetypes'][0]['fields']
for k,f in fields.items():
    if f.get('required'):
        print('REQUIRED:', k, '|', f.get('name'))
"
```

> **Watch out for custom required fields.** The HSE Jira instance had a mandatory `Delivery Area` custom field (`customfield_12452`). Creating issues without it returned a 400. Always check required fields first.

### Creating the hierarchy

For projects that run outside the team's sprint cadence, use the Initiative → Epic → Story hierarchy if your Jira instance supports it:

```python
# Create Initiative
initiative = post("/issue", {"fields": {
    "project": {"key": "YOURPROJECT"},
    "issuetype": {"id": "INITIATIVE_TYPE_ID"},
    "summary": "Your Initiative Name",
    "customfield_XXXXX": {"id": "DELIVERY_AREA_ID"},  # any required custom fields
}})

# Create Epics parented to Initiative
epic = post("/issue", {"fields": {
    "project": {"key": "YOURPROJECT"},
    "issuetype": {"id": "EPIC_TYPE_ID"},
    "summary": "Epic 1: ...",
    "parent": {"key": initiative["key"]},
}})

# Create Stories parented to Epics
story = post("/issue", {"fields": {
    "project": {"key": "YOURPROJECT"},
    "issuetype": {"id": "STORY_TYPE_ID"},
    "summary": "Story 1.1: ...",
    "parent": {"key": epic["key"]},
}})
```

### Reordering stories

To set the execution order (pilot domains first):

```python
# PUT not POST for the agile rank endpoint
req = urllib.request.Request(
    "https://yourorg.atlassian.net/rest/agile/1.0/issue/rank",
    data=json.dumps({"issues": ["PROJ-123"], "rankBeforeIssue": "PROJ-124"}).encode(),
    headers=HEADERS,
    method="PUT"
)
```

---

## 11. Lessons learned

### 1. Verify live infrastructure before planning — AI-generated maps have significant gaps

The AI-generated Front Door routing map we started with was missing multi-region Container App replicas, had decommissioned App Services listed as active, identified 3 Front Door profiles when there were 7, and contained wrong CloudFront distribution IDs.

**The fix:** Pull from the live system using `az afd` CLI commands before writing a single line of planning. Budget half a day for this on a project with many origins.

### 2. The architecture spine is worth the investment

Spending time on the architecture spine before story creation pays back on every single story. The key decisions for Terraform/Terragrunt that most often get missed:

- Exactly which resource naming pattern (singleton `this` vs `for_each` map key)
- Whether DNS edits are in-place or create/destroy (the answer should always be in-place for live systems)
- `lifecycle { prevent_destroy = true }` — when it is required vs optional
- The exact `host_header` vs `origin_host` distinction for Cloudflare origin rules (never both on the same rule)
- `steering_policy` value for load balancers — `"off"` (ordered failover) vs `"geo"` are materially different in behaviour

Get these wrong in the spine and every story inherits the mistake.

### 3. Pre-commit hooks bite on external skill files

When you install skills from npm, the shell scripts in the skill packages may not pass `shellcheck`. The `cc-devops-skills` `terraform-validator` package had SC2155 warnings. Since these are not your files, exclude `.agents/` from shellcheck:

```yaml
# .pre-commit-config.yaml
- id: shellcheck
  exclude: ^\.agents/
```

Similarly, some skill files need trailing whitespace fixes and line ending normalisation. Just let pre-commit fix them on first commit and re-stage.

### 4. BMAD has no native DevOps agent — fill the gap with external skills

The generic `bmad-agent-dev` is a software engineer. It will write Terraform but it doesn't know Terragrunt patterns, Cloudflare provider v5 resource schemas, or what `http_request_origin` phase means.

Install the external skills from skills.sh (see [section 8](#8-terraform-skills)). They cover the general Terraform/Terragrunt correctness. `project-context.md` covers your repo specifics. You need both.

There is an open upstream issue to add a native DevOps persona to BMAD: https://github.com/bmad-code-org/BMAD-METHOD/issues/2187 — worth watching.

### 5. Write issues for the agent the way you'd write a runbook

The quality of the agent's output is directly proportional to the quality of the issue. If you write "update DNS for hse.ie" you get a guess. If you write the exact FQDN it should change from, the exact FQDN it should change to, the Terraform map key, the lifecycle requirement, the NSG gate condition, and the expected curl output after apply — you get correct Terraform.

Time spent writing a good issue saves more time in PR review cycles. For DNS and routing changes on live infrastructure this is not optional.

### 6. Scope the pilot domain correctly

Don't prove the agentic workflow on your most complex domain. Pick something that:
- Has its own separate Front Door profile (contained blast radius)
- Has no clinical or emergency content (no CAB gate, no clinical hours constraint)
- Has a simple structure (one or two Container Apps, one blob origin)
- Is not `www.` anything (no reputational risk if smoke tests catch a problem)

For the HSE migration: `vaccine.hse.ie` was already migrated manually (proof of concept), then `healthpromotion.ie` was chosen as the first agentic pilot — separate profile, single-region, no CAB gate, simple Django/Next.js structure.

### 7. Plan for the process gates, not just the technical ones

The hardest blockers in an infrastructure migration are not technical. They are:
- CAB approval lead time (allow 1–2 weeks minimum for PRD changes)
- Azure team NSG changes in a separate repo (coordinate explicitly — the agent can write the Cloudflare side but cannot merge the Azure side)
- DNS authority for cross-org domains (`.gov.ie` DNS zones require DPER coordination — identify these early)
- Clinical hours constraints (agree maintenance windows with operations before planning the cutover schedule)

None of these appear in Terraform. All of them appear in the acceptance criteria of the stories that need them.

### 8. Jira custom required fields will silently fail

The Jira API returns a 400 with a helpful error message if a required custom field is missing. Always check required fields before bulk-creating issues. In the HSE Jira instance the mandatory field was `Delivery Area` — not something you'd know existed without querying the API first.

### 9. The rollback PR is as important as the cutover PR

Every DNS cutover story should have a rollback PR — a Terraform change that reverts the CNAME content back to the Front Door FQDN — reviewed, approved, and ready to merge before the cutover window opens. The constraint is: rollback must be executable in under 10 minutes.

Test this. A rollback PR that hasn't been planned before the cutover window is not actually a rollback — it's a hope.

---

## 12. Quick reference

### BMAD commands

All BMAD skills are run **inside your AI agent tool** (OpenCode, Claude Code, GitHub Copilot Chat) — not from the terminal. Either use the slash command if your tool supports it, or type the skill name in a chat message.

```
/bmad-generate-project-context   Generate project-context.md from existing repo
/bmad-prd                        Create/update/validate a PRD
/bmad-architecture               Create architecture spine
/bmad-create-epics-and-stories   Break requirements into epics and stories
/bmad-sprint-planning            Generate sprint-status.yaml
/bmad-create-story               Create implementation-ready story file
/bmad-dev-story                  Execute a story (agent implements it)
/bmad-code-review                Code review a PR
/bmad-help                       What to do next
```

For tools that don't support slash commands, just say:

```
Run bmad-prd
```

or

```
Invoke the bmad-architecture skill
```

### Azure Front Door CLI — useful commands

```bash
# List all profiles
az afd profile list --query "[].{name:name, rg:resourceGroup}" -o table

# List origin groups for a profile
az afd origin-group list \
  --profile-name <profile> --resource-group <rg> \
  --query "[].name" -o tsv

# List enabled origins in a group
az afd origin list \
  --profile-name <profile> --resource-group <rg> \
  --origin-group-name <group> \
  --query "[?enabledState=='Enabled'].{name:name, host:hostName}" -o table

# List all routes
az afd route list \
  --profile-name <profile> --resource-group <rg> \
  --endpoint-name <endpoint> -o json

# List custom domains
az afd custom-domain list \
  --profile-name <profile> --resource-group <rg> \
  --query "[].{host:hostName, state:domainValidationState}" -o table
```

### Smoke test after DNS cutover

```bash
# Should see server: cloudflare, no x-azure-fdid
curl -sI https://your-domain.ie/ | grep -E "HTTP|server|x-azure-fdid"

# Check CF-Cache-Status on assets
curl -sI https://assets.your-domain.ie/some-asset.js | grep -E "CF-Cache-Status|Cache-Control"

# Verify Front Door is receiving zero traffic (run after 1 hour)
# Check Azure Monitor / Front Door access logs
```

### Cloudflare origin rules — the key distinction

```hcl
# Single-origin hostname — rewrite Host header only
{
  description = "example.ie catch-all"
  expression  = "(http.host eq \"example.ie\")"
  host_header = "ca-example-euno-prd.northeurope.azurecontainerapps.io"
  # No origin_host — DNS CNAME already points there
}

# Path-split rule — override origin AND host header
{
  description = "example.ie API paths"
  expression  = "(http.host eq \"example.ie\" and starts_with(http.request.uri.path, \"/api/\"))"
  origin_host = "ca-example-api-euno-prd.northeurope.azurecontainerapps.io"
  origin_port = 443
  # No host_header on this rule — never both on the same rule
}
```

### Jira API quick reference

```python
import urllib.request, json, base64, os

BASE = "https://yourorg.atlassian.net/rest/api/3"
AUTH = base64.b64encode(
    f"you@company.com:{os.environ['JIRA_API_TOKEN']}".encode()
).decode()
HEADERS = {
    "Authorization": f"Basic {AUTH}",
    "Content-Type": "application/json",
    "Accept": "application/json"
}

def post(path, body):
    req = urllib.request.Request(
        f"{BASE}{path}",
        data=json.dumps(body).encode(),
        headers=HEADERS,
        method="POST"
    )
    with urllib.request.urlopen(req) as r:
        return json.loads(r.read())

def put(path, body):
    req = urllib.request.Request(
        f"{BASE}{path}",
        data=json.dumps(body).encode(),
        headers=HEADERS,
        method="PUT"
    )
    with urllib.request.urlopen(req) as r:
        return r.status
```

---

## Contributing

If you find something wrong, out of date, or missing — raise a PR. This playbook is meant to be a living document updated as we learn more from running agentic workflows on real infrastructure.

Particular gaps worth filling:
- Multi-cloud examples (AWS Route 53, GCP Cloud Armor)
- Cloudflare Tunnels / Zero Trust setup with agents
- Automated smoke test frameworks for infrastructure cutovers
- Agent workflow for Terraform module development (not just deployment)
