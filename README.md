# docs-actions

This repo contains actions for building and deploying, hugo and sphinx based documentation websites for NGINX.

The action pushes to an Azure Storage Container, and assumed an existing reverse proxy (and or CDN) to make the sites public.

By default, it will also purge the provided CDN path.

## Hugo theme version

By default, this action will pull the latest _released_ hugo theme from https://github.com/nginxinc/nginx-hugo-theme.

If the `force_hugo_theme_version` is set, it will use the tag, branch or commit specified.


## How-to
These instructions apply only to NGINX GitHub doc repositories.
1. Navigate to the actions section of your repository, for example https://github.com/yourOrg/yourRepo/actions
1. On the left side of the page, select Build and deploy (docs). If you don't see this option, follow the [Caller Example](#caller-example).
![Build and deploy (docs)](/images/build-and-deploy.png "Build and deploy (docs)")
1. On the right side, you will see a Run workflow button.
![Run Workflow](/images/run-workflow.png "Run Workflow")
1. Select Run workflow. This opens a menu which lets you select the branch you want to build, and which environment you'd like to deploy to
1. The preview environment generates a custom URL for your preferred branch, and will be available in the workflow output.
All builds, regardless of environment, prints an URL to the summary.
![Summary](/images/summary.png "Summary")

_note: autodeploy from branch is currently in progress. This docs will be updated when available._

## Usage

_note: This action **will not work** with Pull Requests on forked repositories. GitHub does not allow secrets to be shared with forked Pull Requests. Secrets are required as part of Azure interactions. The "Checks and variables" job stops this action from running on forked repositories._

### Security considerations
`AZURE_CREDENTIALS` need to contain service principal credentials with the following roles assigned;
- **Storage Blob Data Contributor**
- **Key Vault Secrets User**
- **CDN Endpoint Contributor**

The service principal (or managed identity) references in the AZURE_CREDENTIALS should have _only_ the roles
listed above. Assigning any more roles is a potential security risk.

`AZURE_KEY_VAULT` should contain the name of a key vault, which contains the following secrets;
- resourceGroupName
- cdnProfileName
- cdnName
- accountName

The KeyVault should have _only_ these secrets, and nothing else, as other secrets could potentially
be exposed.

### Prerequisites
The secrets in AKV allow variables to be shared by all repos contributing to the same base doc sites (as a single source of truth), 
without needing to duplicate github secrets or variables across different github organisations.

Built files are pushed to a location in Azure Container Storage based on the repository name and deploy type.

For example;

A repo called `foo/bar`;
- Preview pushed to `$web/foo/bar/preview/${prNumber}`
- Production build to `$web/foo/bar/latest`



### Caller example

Place this workflow action in the caller repository

``` yml
name: Build and deploy (foobar)
on:
  # Set up the UI in Actions tab for manual builds.
  workflow_dispatch:
    inputs:
      environment:
        description: 'Determines where build gets pushed to'
        required: false
        default: 'preview'
        type: choice
        options:
        - preview
        - dev
        - staging
        - prod

  # Trigger preview builds on pull requests
  pull_request:
    branches:
      - "*"

  # Used for auto deploy builds. This branch _must_ match auto_deploy_branch
  push:
    branches:
      - "main"

jobs:
  # Configure the build
  call-docs-build-push:
    uses: nginxinc/docs-actions/.github/workflows/docs-build-push.yml@main
    with:
      production_url_path: "/foobar/"
      preview_url_path: "/previews/foobar/"
      docs_source_path: "public"
      docs_build_path: "./"
      doc_type: "hugo"
      environment: ${{inputs.environment}}
      # This means, any time there's a push to main, a deployment will automatically be made to dev.
      auto_deploy_branch: "main"
      auto_deploy_env: "dev"
    secrets:
      AZURE_CREDENTIALS: ${{secrets.AZURE_CREDENTIALS}}
      AZURE_KEY_VAULT: ${{secrets.AZURE_KEY_VAULT}}
```

Each docs repo has slightly different requirements in terms of inputs, primarily:
```yml
    with:
      # Where artifacts are sourced after being built
      docs_source_path: "./outputDirOfBuiltDocs/"

      # Defaults to 'hugo', can be set to 'sphinx' for Unit
      doc_type: "hugo"

      # Use / for docs, /nginx-gateway-fabric for gateway fabric
      production_url_path: "/product"

      # Preview URLs should start with /previews/product
      preview_url_path: "/previews/product/"

      # Or /* for docs
      cdn_content_path: "/product"
```
The action should only be triggered if there are changes made in a docs related directory. This can be implemented by setting a `path` value in the actions `on` object.
```yml
on:
  push:
    branches:
      - main
    paths:
      - docsDirectory/**
.......
```

## nginx.org branch

The `nginx.org` branch contains additional workflows and actions for building and deploying the [nginx.org](https://nginx.org) website to AWS S3.

### az-sync action

**Path:** `.github/actions/az-sync/action.yml`

A reusable composite action that logs into Azure, retrieves secrets from an Azure Key Vault, and exports them as environment variables for use in subsequent workflow steps. After all secrets are synced, it logs out of Azure automatically.

#### Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `az_client_id` | Azure Client ID (for OIDC federated login) | Yes | — |
| `az_tenant_id` | Azure Tenant ID | Yes | — |
| `az_subscription_id` | Azure Subscription ID | Yes | — |
| `keyvault` | Name of the Azure Key Vault to read secrets from | Yes | — |
| `secrets-filter` | Comma-separated list of secret name patterns to sync | Yes | `*` |

#### Usage example

```yml
- name: Get Secrets from Azure Key Vault
  uses: nginxinc/docs-actions/.github/actions/az-sync@nginx.org
  with:
    az_client_id: ${{ secrets.AZURE_VAULT_CLIENT_ID }}
    az_tenant_id: ${{ secrets.AZURE_VAULT_TENANT_ID }}
    az_subscription_id: ${{ secrets.AZURE_VAULT_SUBSCRIPTION_ID }}
    keyvault: ${{ secrets.DOCS_VAULTNAME }}
    secrets-filter: 'MySecret1,MySecret2'
```

Each matched secret is exported as an environment variable named after the secret (e.g. `MySecret1`). Multiline secret values are handled using the heredoc syntax supported by `$GITHUB_ENV`.

---

### nginx.org-make-aws workflow

**Path:** `.github/workflows/nginx.org-make-aws.yml`

A reusable (`workflow_call`) workflow that builds the nginx.org website using `make` and deploys it to AWS S3. It supports two separate jobs controlled by the `deployment_env` input:

- **`build-staging`** — Builds the site from source and syncs the output to a versioned staging path in S3 (`staging/<sha>/`). Also uploads a `.deployed.txt` marker file used by the production job.
- **`build-prod`** — Waits for the staging marker to be present for the current commit SHA, then promotes the staged build to the production S3 path (`prod/`).

Both jobs use the [az-sync](#az-sync-action) action to retrieve AWS credentials from Azure Key Vault before assuming an AWS IAM role via OIDC.

#### Secrets

| Secret | Description | Required |
|---|---|---|
| `AZURE_VAULT_CLIENT_ID` | Azure Client ID for Key Vault access | Yes |
| `AZURE_VAULT_SUBSCRIPTION_ID` | Azure Subscription ID | Yes |
| `AZURE_VAULT_TENANT_ID` | Azure Tenant ID | Yes |
| `DOCS_VAULTNAME` | Name of the Azure Key Vault containing AWS credentials | Yes |

The Key Vault referenced by `DOCS_VAULTNAME` must contain the following secrets:

| Key Vault secret | Description |
|---|---|
| `NginxOrgAwsAccountID` | AWS account ID used to construct the IAM role ARN |
| `NginxOrgAwsRoleName` | AWS IAM role name to assume via OIDC |
| `NginxOrgAllowedUsers` | Comma-separated list of GitHub usernames allowed to trigger production deployments |

#### Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `deployment_env` | Target environment: `staging` or `prod` | No | `staging` |
| `url_prod` | Public hostname for the production site | No | `nginx.org` |
| `url_staging` | Public hostname for the staging site | No | `staging.nginx.org` |
| `s3_bucket` | S3 bucket name for deployments | No | `nginx-org-staging` |
| `aws_region` | AWS region for S3 operations | No | `eu-central-1` |

#### Access controls

Both jobs verify that the workflow is triggered from an allowed context before proceeding:

- **Organization**: `nginx` or `nginxinc`
- **Event**: `push` or `workflow_dispatch`
- **Ref** (prod only): `refs/heads/main`
- **Actor** (prod only): must be listed in the `NginxOrgAllowedUsers` Key Vault secret

#### Caller example

```yml
name: nginx.org build and deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      deployment_env:
        description: 'Target environment'
        required: false
        default: staging
        type: choice
        options:
          - staging
          - prod

jobs:
  call-nginx-org-build:
    uses: nginxinc/docs-actions/.github/workflows/nginx.org-make-aws.yml@nginx.org
    with:
      deployment_env: ${{ inputs.deployment_env || 'staging' }}
    secrets:
      AZURE_VAULT_CLIENT_ID: ${{ secrets.AZURE_VAULT_CLIENT_ID }}
      AZURE_VAULT_SUBSCRIPTION_ID: ${{ secrets.AZURE_VAULT_SUBSCRIPTION_ID }}
      AZURE_VAULT_TENANT_ID: ${{ secrets.AZURE_VAULT_TENANT_ID }}
      DOCS_VAULTNAME: ${{ secrets.DOCS_VAULTNAME }}
```
