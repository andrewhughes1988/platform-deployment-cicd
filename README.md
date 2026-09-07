# Central CI/CD Workflows (`platform-deployment-cicd`)

This repository serves as the single source of truth for **all Terraform deployment pipelines and quality guardrails** across the enterprise.

Workload and Platform repositories invoke these reusable workflows via GitHub's `workflow_call`, ensuring strict consistency for formatting, linting, security scanning (Trivy), state locking timeouts, and Azure OIDC authentication.

---

## The Cryptographic Security Lock (Azure OIDC `job_workflow_ref`)

To guarantee that developers **cannot bypass security scanners or write their own unauthorized deployment scripts** (even on standard GitHub Team or Free accounts), authentication is enforced at the Azure boundary.

### How Azure Verifies the Workflow
When GitHub Actions requests an Azure access token using Workload Identity Federation (OIDC), Microsoft Entra ID inspects the token's claims. The token contains a `job_workflow_ref` claim pointing to the exact path of the workflow that initiated the run.

In your Azure App Registration or User-Assigned Managed Identity, add a **Federated Credential** with the following Subject Identifier:

```
repo:andrewhughes1988/<calling-repo-name>:job_workflow_ref:andrewhughes1988/platform-deployment-cicd/.github/workflows/terraform-pipeline.yml@refs/heads/main
```

### Result:
- If a developer uses the approved stub calling `platform-deployment-cicd`, Azure validates the claim and grants access.
- If a developer creates a rogue YAML script in their repo trying to run `terraform apply` directly or omit security scans, Azure sees that `job_workflow_ref` points to the rogue file and **instantly rejects authentication with `AADSTS70021`**.

---

## Branch Protection & CODEOWNERS Setup (GitHub Team / Free)

To prevent developers from tampering with the 10-line caller stub in their repositories:

1. Add a `.github/CODEOWNERS` file in every caller repo:
   ```
   .github/workflows/**   @andrewhughes1988
   ```
2. Enable standard GitHub **Branch Protection** on `main`:
   - [x] **Require a pull request before merging**
   - [x] **Require review from Code Owners**
   - [x] **Require status checks to pass before merging** (Select: `Validate & Plan`)

Developers can modify infrastructure code freely, but cannot merge any pipeline modifications without platform administrator approval.

---

## Available Workflows

### [`terraform-pipeline.yml`](./.github/workflows/terraform-pipeline.yml)

Provides complete pull-request speculative planning and merge-driven deployment:
1. **Quality Gates**: `terraform fmt -check`, `tflint`, and `trivy` static analysis.
2. **Azure OIDC Login**: Secure token exchange (`azure/login@v2`).
3. **Speculative Planning**: Generates plan with `-lock-timeout=10m` and posts summary comments to PRs.
4. **Controlled Apply**: Applies plan artifact on merge to `main` with environment gates.

---

## How Caller Repositories Invoke This Pipeline

In any application or platform repository, place this lightweight 20-line stub at `.github/workflows/deploy.yml`:

```yaml
name: Deploy Infrastructure

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  terraform:
    uses: andrewhughes1988/platform-deployment-cicd/.github/workflows/terraform-pipeline.yml@main
    with:
      environment: dev
      working_directory: "."
      backend_config_file: backend/dev.backend.tfvars
      var_file: environments/dev.tfvars
      azure_client_id: ${{ vars.AZURE_CLIENT_ID_APP_DEV }}
      azure_tenant_id: ${{ vars.AZURE_TENANT_ID }}
      azure_subscription_id: ${{ vars.AZURE_SUBSCRIPTION_ID_APP_DEV }}
```

---

## How Environment Approvals Work (Delegated Caller Approvals)

A common question in reusable workflows is: *Who approves production deployments? Does the central CI/CD team have to approve everything?*

### Caller-Evaluated Environments
Because the workflow uses `environment: ${{ inputs.environment }}` inside the reusable `apply` job, GitHub evaluates environment protection rules **inside the caller repository** (e.g., `azure-platform-core` or `app-order-service`):

1. **Autonomous App Governance**: In `app-order-service`, the application team goes to **Settings** &rarr; **Environments** &rarr; `prod` and assigns their Tech Lead and Ops liaison as required reviewers.
2. **Autonomous Platform Governance**: In `azure-platform-core`, Platform Ops sets their own senior platform engineers as approvers for `prod`.
3. **No Central Bottleneck**: Central CI/CD administrators are **not** spammed or required to manually approve applications they do not own.

---

## Rogue Repository Defense (Why Teams Cannot Bypass Approvals)

*Could a rogue developer create a new repo, copy the caller stub, set themselves as the approver, and deploy to production?*

**No.** Azure Entra ID enforces **Workload Identity Federation Subject Validation**:
1. Every Azure Managed Identity / Service Principal requires an explicit Federated Credential mapped to a specific repository:
   ```text
   repo:andrewhughes1988/<calling-repo-name>:job_workflow_ref:andrewhughes1988/platform-deployment-cicd/.github/workflows/terraform-pipeline.yml@refs/heads/main
   ```
2. If a developer creates an unauthorized repository on GitHub, Azure has **no federated credential** registered for that repo name.
3. When the rogue repo's workflow requests an Azure access token, Entra ID denies the request with `AADSTS70021`. The pipeline fails before executing any Terraform commands.

---

## Golden Template Vending (`app-template-repo`)

To onboard new applications securely:
1. Teams create their repository using [`app-template-repo`](https://github.com/andrewhughes1988/app-template-repo) ("Use this template").
2. The template comes pre-packaged with `.github/CODEOWNERS` and `.github/workflows/deploy.yml`.
3. Platform Ops performs 2 onboarding actions:
   - Enables Branch Protection on `main` (Require PR + Code Owner review + Status check `Validate & Plan`).
   - Adds the repo's federated subject identifier in Azure Entra ID.

