# Audit Controls & Standards for Public Tenders — DEPT® UKI Position

> **Date:** 2026-07-20
> **Author:** gh-service-dept-uki (via Copilot agent)
> **Scope:** Public sector procurement, tender compliance, IaC audit trails
> **Related:** [Stripe Terraform Pattern Assessment](./stripe-terraform-assessment.md)

---

## Executive summary

Public sector tenders — particularly in Ireland, UK, and EU — frequently require evidence of formal quality and security management systems. DEPT® does not hold ISO 9001 (Quality Management) as a company-wide standard. Some agencies hold both, but DEPT® has focused on **ISO 27001** as it is more relevant to digital services, hosting, data protection, and client assurance.

This document outlines:
1. DEPT®'s current certifications and how they map to tender requirements
2. How IaC (Terraform) patterns provide auditable evidence of quality controls
3. How the BMAD/FORGE agentic workflow satisfies audit trail requirements
4. Recommended tender response language for common standards questions

---

## DEPT® certifications and positioning

### What we hold

| Certification | Scope | Relevance to public tenders |
|---|---|---|
| **ISO 27001** | Information Security Management | Security controls, access management, data protection, incident response. Directly relevant to hosting, SaaS delivery, and data processing for public sector clients. |
| **B Corp** | Governance, Transparency, Social Responsibility | Demonstrates audited standards in corporate governance, environmental impact, and stakeholder accountability. Increasingly referenced in social value scoring. |

### What we don't hold

| Standard | Why not | Mitigation |
|---|---|---|
| **ISO 9001** (Quality Management) | DEPT® focused on 27001 as more relevant to digital services. Some acquired agencies may hold it individually. | Reference 27001 process rigour + B Corp governance + delivery frameworks (below) |
| **ISO 14001** (Environmental) | Covered under B Corp environmental pillar | Reference B Corp certification score |
| **ISO 20000** (IT Service Management) | Operational overlap with 27001 + ITIL practices | Reference ITIL-based support frameworks |

### Recommended tender response — ISO 9001 question

> DEPT® does not currently hold ISO 9001 as a company-wide certification. However, we maintain rigorous quality management through:
>
> - **ISO 27001 certification** — demonstrating audited security and process controls across information management, change management, and continuous improvement
> - **B Corp certification** — independently audited standards in governance, transparency, operational practices, and social responsibility
> - **Proven quality management processes** — agile delivery frameworks (Scrum/Kanban with sprint retrospectives), dedicated QA teams, ITIL-based support models, and formal client governance (steering committees, SLA reporting, RAID logs)
>
> We are happy to provide evidence of these controls and discuss how they address the specific quality assurance requirements of this tender.

---

## How IaC provides audit evidence for tenders

### The problem tenders are trying to solve

Public tender evaluation criteria around "quality management" typically probe:

1. **Change control** — can you show who changed what, when, and why?
2. **Traceability** — can you link a deployed change back to a requirement?
3. **Reproducibility** — can you recreate an environment deterministically?
4. **Separation of duties** — who writes vs who approves vs who deploys?
5. **Rollback capability** — can you undo a change safely?
6. **Audit trail** — is there an immutable record of all changes?

### How Terraform + Git answers all six

| Tender requirement | Evidence from Terraform/Git workflow |
|---|---|
| **Change control** | Every infrastructure change is a PR with diff, reviewer approval, and merge commit |
| **Traceability** | PR description links to Jira/GitHub issue → story → epic → PRD requirement |
| **Reproducibility** | `terraform apply` produces identical infrastructure from the same `.tf` files every time |
| **Separation of duties** | Branch protection rules: author ≠ approver. GitHub Environments require separate approval for production apply |
| **Rollback capability** | `git revert` + `terraform apply` restores previous state. Rollback PRs prepared before cutover (BMAD gate) |
| **Audit trail** | Git history is immutable. GitHub audit log records who approved what. Terraform state versioning shows state at any point in time |

### The Stripe Terraform pattern as a concrete example

The [Stripe Terraform pattern](./stripe-terraform-assessment.md) demonstrates this for payment infrastructure:

```
Requirement (Jira story)
    → Agent writes .tf files (traceable to story)
    → PR opened (change control)
    → terraform plan reviewed (separation of duties)
    → Approved by different person (separation)
    → Applied to sandbox first (reproducibility)
    → Promoted to livemode (audit trail)
    → Rollback PR ready (rollback capability)
```

This workflow is the same for DNS records, Cloudflare rules, Azure resources, or Stripe products. The audit evidence is identical regardless of what the Terraform manages.

---

## BMAD/FORGE workflow as quality evidence

### Mapping BMAD to quality management principles

ISO 9001 is built on seven quality management principles. Here's how BMAD addresses each:

| ISO 9001 Principle | BMAD/FORGE Equivalent |
|---|---|
| **Customer focus** | PRD starts with client requirements; acceptance criteria define "done" from client perspective |
| **Leadership** | Architecture spine sets direction; human gates ensure leadership approval at key decisions |
| **Engagement of people** | Sprint planning involves team; PR reviews distribute knowledge |
| **Process approach** | Defined workflow: PRD → Architecture → Stories → Implementation → Review → Apply |
| **Improvement** | Sprint retrospectives; lessons learned (Section 11 of this playbook); architecture spine updates |
| **Evidence-based decision making** | Infrastructure data pulled from live systems (Section 5); never trust AI-generated maps |
| **Relationship management** | Jira integration for client visibility; steering committee governance |

### Mapping FORGE constraints to audit controls

FORGE's `.forge-constraints.md` system provides:

- **Immutable design constraints** — locked decisions that cannot be overridden without explicit approval
- **Constraint compliance tables** — every architecture document states which constraints it honours/violates
- **Red-Team Critic review** — adversarial review catches gaps before delivery
- **Consistency QA** — automated style and naming enforcement

This maps to ISO 27001 Annex A controls:

| ISO 27001 Control | FORGE Implementation |
|---|---|
| A.8.1 Asset management | Infrastructure map in `docs/map/` — committed, versioned |
| A.8.9 Configuration management | Terraform state + `.tf` files = single source of truth |
| A.8.25 Secure development lifecycle | FORGE workflow enforces design → review → implement → validate |
| A.8.32 Change management | PR workflow with branch protection, required reviews, status checks |
| A.5.8 Information security in project management | `.forge-constraints.md` enforces security controls from project inception |

---

## Public tender compliance matrix

### Common tender requirements and DEPT® evidence

| Tender Requirement | Standard Referenced | DEPT® Evidence |
|---|---|---|
| Information security management system | ISO 27001 | ✅ Certified |
| Quality management system | ISO 9001 | ⚠️ Not certified — mitigate with 27001 + B Corp + delivery frameworks |
| Environmental management | ISO 14001 | ⚠️ Not certified — mitigate with B Corp environmental score |
| Data protection compliance | GDPR / DPA 2018 | ✅ Covered under 27001 + DPO + Data Processing Agreements |
| Business continuity | ISO 22301 | ⚠️ Not certified — mitigate with 27001 BCM controls + DR runbooks |
| Accessibility | WCAG 2.2 AA / EN 301 549 | ✅ Axe/Pa11y automated testing + manual audits + Storybook a11y addon |
| Secure development | OWASP / NIST | ✅ SonarCloud SAST + dependency scanning + tfsec + Checkov (for IaC) |
| Change management process | ITIL | ✅ PR-based workflow + GitHub Environments + approval gates |
| Disaster recovery | N/A | ✅ Terraform reproducibility + state backups + rollback PRs |
| Supply chain security | Cyber Essentials / NIS2 | ✅ Dependabot + SBOM generation + signed commits |

### Ireland-specific (OGP / eTenders)

For Office of Government Procurement tenders:

| OGP Requirement | DEPT® Response |
|---|---|
| DPER ICT Security Standards | ISO 27001 certification + Cloudflare WAF + Azure/AWS native security |
| Public Spending Code compliance | Tracked via Jira + sprint velocity reporting + FinOps dashboards |
| Irish language provision | Contentful i18n support + configurable locale routing |
| GDPR / Data Protection Commission alignment | DPO appointed + DPIAs conducted per project + data residency (EU-West-1 / North Europe) |

### UK-specific (CCS / Digital Marketplace)

For Crown Commercial Service frameworks:

| CCS Requirement | DEPT® Response |
|---|---|
| Cyber Essentials Plus | ✅ (required for many CCS lots) |
| G-Cloud / DOS compliance | ✅ Service definitions + pricing schedules published |
| GDS Service Standard | ✅ User-centred design process + accessibility compliance |
| SC/DV clearance | ⚠️ Case-by-case — confirm with HR for specific personnel |

---

## Recommendations

### For tender responses

1. **Never claim ISO 9001** — state clearly we don't hold it, then demonstrate equivalent controls
2. **Lead with ISO 27001** — it subsumes most quality controls relevant to digital delivery
3. **Reference B Corp** — increasingly valued in social value scoring (often 10-20% of tender weighting)
4. **Provide evidence, not promises** — link to Git history, PR workflows, Terraform plans as concrete artefacts
5. **Offer a controls mapping** — show evaluators exactly how your processes map to their requirements

### For project delivery (proving the claim)

1. **Use Terraform for all manageable infrastructure** — creates immutable audit trail automatically
2. **Use BMAD workflow for agent-driven work** — creates traceable requirement → implementation → verification chain
3. **Maintain FORGE constraints** — locked constraints = auditable policy enforcement
4. **Keep infrastructure maps in Git** — `docs/map/` committed and versioned = asset management evidence
5. **Run retrospectives and document lessons** — continuous improvement evidence for evaluators
6. **Enable GitHub audit log forwarding** — enterprise audit events → SIEM for compliance reporting

### For the uki-stripe-yste project specifically

If Stripe YSTE adds payment features for event ticketing:
- Use the [Stripe Terraform pattern](./stripe-terraform-assessment.md) to manage Stripe config
- PCI-DSS scope stays minimal (Stripe Checkout / tokenisation)
- Terraform audit trail satisfies payment-related change control requirements
- Webhook signature verification satisfies data integrity controls

---

## Template: Tender response section — Quality & Standards

```markdown
## Quality Management Approach

### Certifications

DEPT® holds the following certifications relevant to this engagement:

- **ISO/IEC 27001:2022** — Information Security Management System
  (Certificate No: [INSERT], Scope: [INSERT])
- **B Corporation** — Independently audited governance, transparency,
  and social responsibility standards (Score: [INSERT])

### Quality Management Framework

While DEPT® does not hold ISO 9001 as a standalone certification, our
delivery quality is assured through:

**Process controls (embedded in ISO 27001):**
- Formal change management via pull request workflows with mandatory peer review
- Infrastructure-as-Code (Terraform) providing reproducible, auditable deployments
- Automated quality gates: linting, type checking, unit/integration testing, security scanning

**Delivery framework:**
- Agile methodology (Scrum/Kanban) with sprint retrospectives driving continuous improvement
- Dedicated QA function with automated accessibility (WCAG 2.2 AA) and performance testing
- ITIL-aligned support model with defined SLAs, incident management, and problem management

**Governance:**
- Client steering committees with defined RACI and escalation paths
- RAID log management with fortnightly client reporting
- B Corp-audited governance ensuring transparency and accountability

### Evidence

We can provide:
- Sample Git audit trails showing full change traceability
- Terraform plan/apply records demonstrating reproducible deployments
- Sprint velocity and burndown reporting
- Client satisfaction survey results and NPS scores
- ISO 27001 Statement of Applicability (on request, under NDA)
```

---

## References

- [ISO 27001:2022](https://www.iso.org/standard/27001) — Information Security Management
- [ISO 9001:2015](https://www.iso.org/standard/62085.html) — Quality Management Systems
- [B Corp Certification](https://www.bcorporation.net/) — DEPT® profile
- [Stripe Terraform Pattern Assessment](./stripe-terraform-assessment.md) — IaC audit trail example
- [OGP eTenders](https://www.etenders.gov.ie/) — Irish public procurement
- [CCS Digital Marketplace](https://www.digitalmarketplace.service.gov.uk/) — UK frameworks
- [GDS Service Standard](https://www.gov.uk/service-manual/service-standard) — UK digital delivery standard
