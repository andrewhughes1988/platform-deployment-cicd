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

In any application or platform repository, place this lightweight 15-line stub at `.github/workflows/deploy.yml`:

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
      working_directory: compute/container-app-environments
      backend_config_file: environments/dev/backend.tfvars
      var_file: environments/dev/terraform.tfvars
      azure_client_id: ${{ vars.AZURE_CLIENT_ID_DEV }}
      azure_tenant_id: ${{ vars.AZURE_TENANT_ID }}
      azure_subscription_id: ${{ vars.AZURE_SUBSCRIPTION_ID_DEV }}
```

