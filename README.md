# Central CI/CD Workflows (`platform-deployment-cicd`)

This repository serves as the single source of truth for **all Terraform deployment pipelines, quality guardrails, and Stategraph execution workflows** across the enterprise.

Workload and Platform repositories invoke these reusable workflows via GitHub's `workflow_call`, ensuring strict consistency for formatting, linting, security scanning (Trivy), Stategraph Velocity execution, and Azure OIDC authentication.

---

## Stategraph Velocity & CI/CD Architecture

This CI/CD ecosystem uses **[Stategraph](https://stategraph.com)** to execute parallel plans and applies with resource-level locking:

```
+-------------------------------------------------------------------------+
| GITHUB ACTIONS RUNNER                                                   |
|                                                                         |
| 1. Environment Injection (Zero Hardcoding):                             |
|    - STATEGRAPH_API_BASE: ${{ vars.STATEGRAPH_API_BASE }}               |
|    - STATEGRAPH_TENANT_ID: ${{ vars.STATEGRAPH_TENANT_ID }}             |
|    - STATEGRAPH_API_KEY: ${{ secrets.STATEGRAPH_API_KEY }}             |
|                                                                         |
| 2. Quality & Security Scans:                                            |
|    - terraform fmt -check                                               |
|    - tflint (Azure ruleset)                                             |
|    - trivy config (CVE & misconfiguration scans)                        |
|                                                                         |
| 3. Azure OIDC Exchange (Federated Credential verified by Entra ID)       |
|                                                                         |
| 4. Stategraph Execution:                                                |
|    - Reads stategraph.json in working-directory                         |
|    - stategraph plan --workspace <env>                                  |
|    - stategraph apply --workspace <env> --auto-approve                   |
+------------------------------------+------------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------------+
| STATEGRAPH CLOUD                                                        |
|                                                                         |
| - Locks only the exact resources being modified (Resource-Level Locking) |
| - Rejects conflicting commits automatically without corrupting state    |
| - Multiple PRs touching disjoint resources apply concurrently           |
+-------------------------------------------------------------------------+
```

### Zero-Hardcoding Configuration
Workflows never hardcode API keys, tenant IDs, or server endpoints:
- **`STATEGRAPH_API_KEY`**: Sourced from GitHub Actions Environment Secrets (isolated between `dev` and `prod`).
- **`STATEGRAPH_API_BASE`**: Sourced from GitHub Actions Variables (`vars.STATEGRAPH_API_BASE`).
- **`STATEGRAPH_TENANT_ID`**: Sourced from GitHub Actions Variables (`vars.STATEGRAPH_TENANT_ID`).

---

## The Cryptographic Security Lock (Azure OIDC `job_workflow_ref`)

To guarantee that developers **cannot bypass security scanners or write their own unauthorized deployment scripts** (even on standard GitHub Team or Free accounts), authentication is enforced at the Azure boundary.

### How Azure Verifies the Workflow
When GitHub Actions requests an Azure access token using Workload Identity Federation (OIDC), Microsoft Entra ID inspects the token's claims. The token contains a `job_workflow_ref` claim pointing to the exact path of the workflow that initiated the run.

In your Azure App Registration or User-Assigned Managed Identity, add a **Federated Credential** with the following Subject Identifier:

```text
repo:andrewhughes1988/<calling-repo-name>:job_workflow_ref:andrewhughes1988/platform-deployment-cicd/.github/workflows/terraform-pipeline.yml@refs/heads/main
```

### Result:
- If a developer uses the approved stub calling `platform-deployment-cicd`, Azure validates the claim and grants access.
- If a developer creates a rogue YAML script in their repo trying to run `terraform apply` directly or omit security scans, Azure sees that `job_workflow_ref` points to the rogue file and **instantly rejects authentication with `AADSTS70021`**.

---

## Branch Protection & CODEOWNERS Setup (GitHub Team / Free)

To prevent developers from tampering with the caller stub in their repositories:

1. Add a `.github/CODEOWNERS` file in every caller repo:
   ```text
   .github/workflows/**   @andrewhughes1988
   ```
2. Enable standard GitHub **Branch Protection** on `main`:
   - [x] **Require a pull request before merging**
   - [x] **Require review from Code Owners**
   - [x] **Require status checks to pass before merging** (Select: `Validate & Plan`)

Developers can modify infrastructure code freely, but cannot merge any pipeline modifications without platform administrator approval.

---

## Available Reusable Workflows

### 1. `terraform-pipeline.yml`
Unified pull-request plan and merge apply workflow for micro-repositories and single-stack projects:
1. **Quality Gates**: `terraform fmt`, `tflint`, and `trivy` static analysis.
2. **Azure OIDC Exchange**: Exchanges GitHub token for Azure federated access.
3. **Stategraph Speculative Plan**: Executes `stategraph plan` and updates a sticky comment on the PR.
4. **Stategraph Apply**: On merge to `main`, executes `stategraph apply --auto-approve` with environment approvals.

### 2. `terraform-plan.yml`
Modular, reusable plan workflow designed for matrix execution in monorepos (`azure-platform-core`). Generates high-level change summaries in `$GITHUB_STEP_SUMMARY` without exposing sensitive resource bodies or uploading plan files to external storage.

### 3. `terraform-apply.yml`
Modular, reusable apply workflow designed for manual gated deployments or post-merge execution in monorepos. Evaluates environment protection rules inside the caller repository and applies changes via Stategraph.

---

## How Caller Repositories Invoke This Pipeline

In any application or platform repository, place this lightweight stub at `.github/workflows/deploy.yml`:

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
      var_file: environments/dev.tfvars
      azure_client_id: ${{ vars.AZURE_CLIENT_ID_APP_DEV }}
      azure_tenant_id: ${{ vars.AZURE_TENANT_ID }}
      azure_subscription_id: ${{ vars.AZURE_SUBSCRIPTION_ID_APP_DEV }}
    secrets:
      STATEGRAPH_API_KEY: ${{ secrets.STATEGRAPH_API_KEY }}
```

---

## Concurrent Pull Requests & Blast Radius

Because Stategraph locks per resource rather than per state file:
- Two PRs modifying disjoint resources (e.g. adding different VMs or different route table entries) plan and apply concurrently.
- If two PRs modify the same resource simultaneously, Stategraph detects the conflicting transaction ID at commit and safely fails the second apply with instructions to re-plan.
- No `force-unlock` commands or serialized queue bottlenecks are ever required.
