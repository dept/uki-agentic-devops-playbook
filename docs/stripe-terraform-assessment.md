# Stripe Terraform Pattern — Assessment for DEPT® UKI Stack

> **Date:** 2026-07-13
> **Source:** [Configuring Stripe using Terraform and AI agents](https://stripe.dev/blog/ai-agents-terraform-stripe-infrastructure) (Stripe Dev Blog, 2026-01-27)
> **Assessed by:** gh-service-dept-uki (via Copilot agent)
> **Repos reviewed:** `dept/uki-stripe-yste`, `dept/emea-agentic-architecture`, `dept/uki-agentic-devops-playbook`

---

## Executive summary

Stripe's blog post advocates using AI agents to **author Terraform code** for Stripe configuration (products, prices, webhooks) rather than letting agents make direct API calls. This pattern aligns directly with the BMAD playbook's "planning first, code second" philosophy and can be implemented using the existing BMAD + FORGE stack already in use across DEPT® EMEA.

**The key insight:** AI agents are stochastic. Terraform is deterministic. Make the agent produce the `.tf` files. Let `terraform plan` and `terraform apply` do the actual work — with human review gates intact.

---

## Repos reviewed

### dept/uki-stripe-yste

| Attribute | Value |
|---|---|
| **Description** | Stripe YSTE 2027 (Young Scientist & Technology Exhibition) website |
| **Stack** | Next.js 16, React 19, TypeScript 5, Contentful CMS, Tailwind v4, shadcn/ui |
| **Hosting** | Vercel (auto-deploy from GitHub) |
| **IaC** | None — Vercel-managed, no Terraform |
| **Stripe payments** | **None** — "Stripe" in the name refers to the Stripe Accelerator programme, not payment processing |
| **CI/CD** | GitHub Actions (ESLint, TypeScript, Vitest, Commitlint, SonarCloud) |

> ⚠️ Despite the name, this repo has **no Stripe SDK, no webhook handlers, and no payment endpoints**. If Stripe payment integration is added (e.g. for event ticket sales or donations), the Terraform pattern becomes directly applicable.

### dept/emea-agentic-architecture

| Attribute | Value |
|---|---|
| **Framework** | FORGE (Framework for Organised Reasoning, Generation & Execution) |
| **Key agents** | Solution Architect (orchestrator), Cloud & DevOps Architect, Discovery Analyst, Red-Team Critic, Consistency QA |
| **IaC guidance** | Terraform (preferred), with constraint enforcement via `.forge-constraints.md` |
| **Terraform patterns** | Module structure, state backend locking, workspaces per environment, CI/CD gates with OIDC auth |

### dept/uki-agentic-devops-playbook (this repo)

| Attribute | Value |
|---|---|
| **Framework** | BMAD (Broad Method for Agentic Development) |
| **Focus** | Existing infrastructure — migrations, cutovers, module changes |
| **IaC** | Terraform + Terragrunt with external agent skills |
| **Human gates** | No `terraform apply` without approval, rollback PRs prepared before cutover |

---

## The Stripe Terraform pattern

### Problem it solves

When AI agents configure Stripe directly via API calls:

1. **No transparency** — API calls are buried in ephemeral agent threads, hard to audit days later
2. **No consistency** — Same prompt twice produces different results, causing drift between sandbox and production
3. **No auditability** — API calls describe transitions, not current state; impossible to reconstruct state at a point in time

### The solution

Make the AI agent a **code author**, not an **operator**:

```
User describes Stripe setup
       ↓
AI agent writes .tf files (products, prices, webhooks)
       ↓
Human reviews PR
       ↓
terraform plan (sandbox)
       ↓
terraform apply (sandbox) — with approval gate
       ↓
Promote to livemode via workspace switch
```

### Core Terraform resources

The [Stripe Terraform provider](https://docs.stripe.com/terraform) (`stripe/stripe ~> 0.1`) supports:

| Resource | Purpose |
|---|---|
| `stripe_product` | Defines products (e.g. "Standard Plan") |
| `stripe_price` | Attaches pricing to products (monthly, yearly, usage-based) |
| `stripe_webhook_endpoint` | Registers webhook URLs and event subscriptions |

### Multi-environment via workspaces

```bash
terraform workspace new sandbox    # Stripe test mode
terraform workspace new livemode   # Stripe live mode

# Each workspace gets its own STRIPE_API_KEY
terraform workspace select sandbox
export STRIPE_API_KEY="sk_test_..."
terraform plan && terraform apply

terraform workspace select livemode
export STRIPE_API_KEY="sk_live_..."
terraform plan && terraform apply
```

---

## How this maps to the current DEPT® stack

### BMAD alignment (this playbook)

The Stripe Terraform pattern maps 1:1 to the BMAD workflow already documented here:

| BMAD Step | Stripe Terraform Equivalent |
|---|---|
| Generate project context | Audit current Stripe Dashboard config — export products, prices, webhooks |
| PRD | Define pricing structure, webhook requirements, environment strategy |
| Architecture spine | Lock Terraform module layout, state backend, workspace strategy, secret management |
| Epics and stories | Break into: "Define product catalogue", "Configure webhook endpoints", "Set up CI/CD pipeline" |
| Agent writes Terraform / opens PR | Agent generates `.tf` files using `stripe/stripe` provider |
| Review plan output | `terraform plan` review — verify no unexpected destroys |
| Human gates | No `terraform apply` without approval; sandbox validated before livemode |
| Smoke tests | Verify Stripe Dashboard reflects expected state; test webhook delivery |

**The BMAD human gates are critical here.** Stripe configuration changes can immediately affect live payments. The playbook's existing gates (no apply without approval, rollback PR prepared) apply directly.

### FORGE alignment (emea-agentic-architecture)

FORGE provides the architectural framework for **new** Stripe Terraform infrastructure:

| FORGE Component | Application |
|---|---|
| **Cloud & DevOps Architect** | Designs Terraform module structure, state backend, CI/CD gates |
| **Constraint enforcement** | `.forge-constraints.md` locks provider version, state backend, secret rotation policy |
| **Discovery Analyst** | Reads Stripe API docs, PCI-DSS requirements, existing Dashboard config |
| **Red-Team Critic** | Reviews for IAM over-privilege, webhook signature validation gaps, PCI scope violations |
| **Consistency QA** | Enforces naming conventions, validates constraint compliance |

#### Recommended `.forge-constraints.md` for Stripe Terraform

```yaml
---
project: stripe-terraform
enforced: true
---

# Hard Constraints

- **IaC Tooling:** Terraform with stripe/stripe provider (locked)
  - No direct Stripe API calls from CI/CD (locked)
  - Module structure: modules/{products,pricing,webhooks,checkout} (locked)

- **State Management:** Remote backend with locking (locked)
  - State encryption required — contains Stripe resource IDs
  - Separate state per workspace (sandbox/livemode)

- **Secrets:** Stripe API keys via environment variables only (locked)
  - Never committed to repository
  - Rotated via secrets manager (AWS Secrets Manager / GitHub Secrets)
  - sk_test_* for sandbox workspace, sk_live_* for livemode workspace

- **Webhook Security:** HMAC-SHA256 signature verification (locked)
  - Webhook signing secret managed as Terraform output, stored in secrets manager
  - Never logged or exposed in plan output

- **PCI-DSS Scope:** Tokenisation required (locked)
  - Raw card data (PAN) never stored in state or config
  - Use Stripe Checkout or Elements for card collection — never raw forms

- **Environment Isolation:** Workspace-based (locked)
  - sandbox = Stripe test mode (sk_test_*)
  - livemode = Stripe production (sk_live_*)
  - No cross-environment state access
```

---

## Implementation plan for dept projects

### Phase 1 — Foundation (FORGE workflow)

Use FORGE to produce the architecture document:

1. **Create project slug** in `emea-agentic-architecture` (e.g. `stripe-terraform`)
2. **Run Discovery Analyst** against Stripe Terraform provider docs and existing Dashboard config
3. **Cloud & DevOps Architect** designs module layout, state backend, CI/CD pipeline
4. **Red-Team Critic** reviews for security gaps (webhook validation, secret exposure, PCI scope)
5. **Lock constraints** in `.forge-constraints.md`

### Phase 2 — Implementation (BMAD workflow)

Use BMAD in the target repo to implement:

1. **Install Terraform skills** (already documented in this playbook):
   ```bash
   npx skills add https://github.com/hashicorp/agent-skills --skill terraform-style-guide --yes
   npx skills add https://github.com/akin-ozer/cc-devops-skills --skill terraform-validator --yes
   ```

2. **Generate project context** — scan existing repo for conventions, add Stripe-specific rules

3. **Create stories** following BMAD issue template:
   - Story 1: Terraform provider setup + state backend
   - Story 2: Product and price resources
   - Story 3: Webhook endpoint resources
   - Story 4: CI/CD pipeline (plan on PR, apply on merge with approval gate)
   - Story 5: Environment promotion (sandbox → livemode)

4. **Assign to Copilot agent** — agent writes `.tf` files, opens PR

5. **Human review gates:**
   - `terraform plan` output reviewed before any apply
   - Sandbox validated before livemode promotion
   - Rollback PR prepared (revert product/price changes)

### Phase 3 — CI/CD pipeline

```yaml
# .github/workflows/stripe-terraform.yml
name: Stripe Terraform

on:
  pull_request:
    paths: ['terraform/stripe/**']
  push:
    branches: [main]
    paths: ['terraform/stripe/**']

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
        working-directory: terraform/stripe
      - run: terraform plan -no-color
        working-directory: terraform/stripe
        env:
          STRIPE_API_KEY: ${{ secrets.STRIPE_TEST_API_KEY }}

  apply:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: stripe-sandbox  # Requires manual approval
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
        working-directory: terraform/stripe
      - run: terraform apply -auto-approve
        working-directory: terraform/stripe
        env:
          STRIPE_API_KEY: ${{ secrets.STRIPE_TEST_API_KEY }}
```

> **Livemode apply** should require a separate GitHub Environment (`stripe-livemode`) with additional reviewers and branch protection.

---

## Applicability to uki-stripe-yste

The YSTE project currently has **no Stripe payment integration**. However, if any of the following features are scoped, this pattern applies directly:

| Potential Feature | Stripe Resources Needed |
|---|---|
| Event ticket sales (YSTE 2027) | `stripe_product`, `stripe_price`, `stripe_webhook_endpoint` |
| Donation collection | `stripe_product` (donation), `stripe_price` (flexible amount) |
| Sponsor portal payments | `stripe_product`, `stripe_price`, `stripe_webhook_endpoint` |
| Alumni membership fees | `stripe_product`, `stripe_price` (recurring), `stripe_webhook_endpoint` |

The existing Vercel + GitHub Actions CI/CD can be extended with a Terraform step for Stripe config. No infrastructure migration needed — Stripe Terraform runs independently of the app hosting.

---

## What gh-service-dept-uki can do

As a service account with PR permissions, the implementation path is:

1. **This PR** — adds this assessment to the playbook for team review
2. **Future PRs** — when a project needs Stripe integration:
   - Generate the Terraform scaffold (provider, backend, module structure)
   - Create GitHub Actions workflow for plan/apply
   - Open PR for human review — never apply without approval
3. **Cannot do:**
   - Run `terraform apply` (requires Stripe API keys)
   - Access Stripe Dashboard (requires human auth)
   - Approve production changes (requires human gate)

---

## References

- [Stripe Terraform Provider Docs](https://docs.stripe.com/terraform)
- [Stripe Dev Blog — AI Agents & Terraform](https://stripe.dev/blog/ai-agents-terraform-stripe-infrastructure)
- [BMAD Method](https://github.com/bmad-code-org/BMAD-METHOD)
- [FORGE Framework](https://github.com/dept/emea-agentic-architecture) (internal)
- [HashiCorp Agent Skills](https://github.com/hashicorp/agent-skills)
