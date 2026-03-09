# docs-actions

This repo contains reusable GitHub Actions workflows for building and deploying Hugo-based documentation websites for NGINX.

## Available workflows

| Workflow file | Description |
|---|---|
| [`docs-build-push.yml`](#docs-build-pushyml-azure) | Builds docs and syncs them to Azure Blob Storage (az-sync) |
| [`docs-build-push-aws.yml`](#docs-build-push-awsyml-aws) | Builds docs and syncs them to AWS S3, using Azure Key Vault for secrets |
| [`nginx.org-make-aws.yml`](#nginxorg-make-awsyml) | Builds and deploys nginx.org to AWS S3 |

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

---

## `docs-build-push.yml` (Azure)

Builds Hugo or Sphinx documentation and pushes artifacts to an Azure Blob Storage container. Assumes an existing reverse proxy (and/or CDN) to make the sites public. By default, it also purges the provided CDN path.

_note: This workflow **will not work** with Pull Requests on forked repositories. GitHub does not allow secrets to be shared with forked Pull Requests. Secrets are required as part of Azure interactions. The "Checks and variables" job stops this workflow from running on forked repositories._

### Security considerations

`AZURE_CREDENTIALS` must contain service principal credentials with the following roles assigned:
- **Storage Blob Data Contributor**
- **Key Vault Secrets User**
- **CDN Endpoint Contributor**

The service principal (or managed identity) referenced in `AZURE_CREDENTIALS` should have _only_ the roles listed above. Assigning any more roles is a potential security risk.

`AZURE_KEY_VAULT` should contain the name of a key vault with the following secrets:
- `resourceGroupName`
- `cdnProfileName`
- `cdnName`
- `accountName`

The Key Vault should have _only_ these secrets, and nothing else, as other secrets could potentially be exposed.

### Prerequisites

The secrets in AKV allow variables to be shared by all repos contributing to the same base doc sites (as a single source of truth), without needing to duplicate GitHub secrets or variables across different GitHub organisations.

Built files are pushed to a location in Azure Blob Storage based on the repository name and deploy type.

For example, a repo called `foo/bar`:
- Preview pushed to `$web/foo/bar/preview/${prNumber}`
- Production build to `$web/foo/bar/latest`

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | No | `preview` | Deployment environment. One of `preview`, `dev`, `staging`, `prod`. |
| `production_url_path` | **Yes** | — | URL path appended to the baseURL for production builds (e.g. `/nginx-agent`). |
| `preview_url_path` | **Yes** | — | URL path appended to the baseURL for PR preview builds (e.g. `/previews/nginx-agent/`). |
| `docs_source_path` | **Yes** | — | Directory of built docs files (e.g. `./public/`). |
| `docs_build_path` | No | `./` | Directory where the Hugo or Sphinx build command should be run from. |
| `doc_type` | No | `hugo` | Type of source docs. Supports `hugo` and `sphinx`. |
| `force_hugo_theme_version` | No | `""` | Overrides the default latest Hugo theme. Must start with `v` (e.g. `v2.1.0`). |
| `auto_deploy_branch` | No | — | Branch that triggers an automatic deployment to `auto_deploy_env`. Requires an `on.push` event for this branch in the caller. |
| `auto_deploy_env` | No | — | Environment to deploy `auto_deploy_branch` to. `preview` is not supported. |
| `node_dep` | No | — | Set to `true` to run `npm ci` before building. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `AZURE_CREDENTIALS` | **Yes** | Azure service principal credentials JSON. |
| `AZURE_KEY_VAULT` | **Yes** | Name of the Azure Key Vault containing deployment secrets. |

### Caller example

Place this workflow file in the caller repository at `.github/workflows/`:

```yml
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
  call-docs-build-push:
    uses: nginxinc/docs-actions/.github/workflows/docs-build-push.yml@main
    with:
      production_url_path: "/foobar/"
      preview_url_path: "/previews/foobar/"
      docs_source_path: "public"
      docs_build_path: "./"
      doc_type: "hugo"
      environment: ${{inputs.environment}}
      # Any push to main will automatically deploy to dev.
      auto_deploy_branch: "main"
      auto_deploy_env: "dev"
    secrets:
      AZURE_CREDENTIALS: ${{secrets.AZURE_CREDENTIALS}}
      AZURE_KEY_VAULT: ${{secrets.AZURE_KEY_VAULT}}
```

Each docs repo has slightly different requirements. Common inputs to customise:

```yml
    with:
      # Where artifacts are sourced after being built
      docs_source_path: "./outputDirOfBuiltDocs/"

      # Defaults to 'hugo', can be set to 'sphinx'
      doc_type: "hugo"

      # Use / for docs, /nginx-gateway-fabric for gateway fabric
      production_url_path: "/product"

      # Preview URLs should start with /previews/product
      preview_url_path: "/previews/product/"
```

The workflow should only be triggered when there are changes in a docs-related directory. Use the `paths` filter in the caller's `on` object:

```yml
on:
  push:
    branches:
      - main
    paths:
      - docsDirectory/**
```

---

## `docs-build-push-aws.yml` (AWS)

Builds Hugo or Sphinx documentation and pushes artifacts to an AWS S3 bucket. AWS credentials are retrieved from Azure Key Vault using federated identity (OIDC), so Azure secrets are still required.

_note: This workflow **will not work** with Pull Requests on forked repositories._

### Security considerations

Federated credentials must be configured in Azure with the following roles assigned to the managed identity:
- **Key Vault Secrets User** (on the Key Vault used for `DOCS_VAULTNAME`)

The Azure Key Vault (referenced by `DOCS_VAULTNAME`) must contain the following secrets:
- `AWSAccount` — AWS account ID
- `AWSRoleName` — IAM role name to assume via OIDC
- `AWSS3Bucket` — S3 bucket name

### Prerequisites

The managed identity must be allowed to assume the specified AWS IAM role via OIDC. The IAM role requires:
- `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` permissions on the target S3 bucket

Built files are pushed to an S3 location based on the repository name and deploy type.

For example, a repo called `foo/bar`:
- Preview pushed to `s3://<bucket>/dev/foo/bar/previews/${prNumber}`
- Production build to `s3://<bucket>/${env}/foo/bar/latest/`

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `environment` | No | `preview` | Deployment environment. One of `preview`, `dev`, `staging`, `prod`. |
| `production_url_path` | **Yes** | — | URL path appended to the baseURL for production builds (e.g. `/nginx-agent`). |
| `preview_url_path` | **Yes** | — | URL path appended to the baseURL for PR preview builds (e.g. `/previews/nginx-agent/`). |
| `docs_source_path` | **Yes** | — | Directory of built docs files (e.g. `./public/`). |
| `docs_build_path` | No | `./` | Directory where the Hugo or Sphinx build command should be run from. |
| `doc_type` | No | `hugo` | Type of source docs. Supports `hugo` and `sphinx`. |
| `force_hugo_theme_version` | No | `""` | Overrides the default latest Hugo theme. Must start with `v` (e.g. `v2.1.0`). |
| `auto_deploy_branch` | No | — | Branch that triggers an automatic deployment to `auto_deploy_env`. Requires an `on.push` event for this branch in the caller. |
| `auto_deploy_env` | No | — | Environment to deploy `auto_deploy_branch` to. `preview` is not supported. |
| `node_dep` | No | — | Set to `true` to run `npm ci` before building. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `AZURE_VAULT_CLIENT_ID` | **Yes** | Client ID of the managed identity used to authenticate with Azure. |
| `AZURE_VAULT_SUBSCRIPTION_ID` | **Yes** | Azure subscription ID. |
| `AZURE_VAULT_TENANT_ID` | **Yes** | Azure tenant ID. |
| `DOCS_VAULTNAME` | **Yes** | Name of the Azure Key Vault containing AWS deployment secrets. |

### Caller example

```yml
name: Build and deploy (foobar)
on:
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

  pull_request:
    branches:
      - "*"

  push:
    branches:
      - "main"

jobs:
  call-docs-build-push-aws:
    uses: nginxinc/docs-actions/.github/workflows/docs-build-push-aws.yml@main
    with:
      production_url_path: "/foobar/"
      preview_url_path: "/previews/foobar/"
      docs_source_path: "public"
      docs_build_path: "./"
      doc_type: "hugo"
      environment: ${{inputs.environment}}
      auto_deploy_branch: "main"
      auto_deploy_env: "dev"
    secrets:
      AZURE_VAULT_CLIENT_ID: ${{secrets.AZURE_VAULT_CLIENT_ID}}
      AZURE_VAULT_SUBSCRIPTION_ID: ${{secrets.AZURE_VAULT_SUBSCRIPTION_ID}}
      AZURE_VAULT_TENANT_ID: ${{secrets.AZURE_VAULT_TENANT_ID}}
      DOCS_VAULTNAME: ${{secrets.DOCS_VAULTNAME}}
```

---

## `nginx.org-make-aws.yml`

Builds and deploys the nginx.org website (using `make`) to AWS S3. Supports a two-stage deploy: staging first, then promotion to production.

### Security considerations

The caller must be in the `nginx` or `nginxinc` GitHub organisation. Production deploys are restricted to commits pushed to `main` by users listed in the `ALLOWED_USERS` secret.

The IAM role specified by `AWS_ROLE_NAME` requires:
- `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket`, `s3:GetObject` permissions on `s3_bucket`

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `deployment_env` | No | `staging` | Deployment environment. One of `staging`, `prod`. |
| `url_prod` | No | `nginx.org` | Public hostname for the production site. |
| `url_staging` | No | `staging.nginx.org` | Public hostname for the staging site. |
| `s3_bucket` | No | `nginx-org-staging` | S3 bucket name. |
| `aws_region` | No | `eu-central-1` | AWS region of the S3 bucket. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `AWS_ACCOUNT_ID` | **Yes** | AWS account ID used to construct the IAM role ARN. |
| `AWS_ROLE_NAME` | **Yes** | IAM role name to assume via OIDC. |
| `ALLOWED_USERS` | **Yes** | Space-separated list of GitHub usernames allowed to trigger production deploys. |

### Caller example

```yml
name: Deploy nginx.org
on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      deployment_env:
        description: 'Deployment environment'
        required: false
        default: 'staging'
        type: choice
        options:
        - staging
        - prod

jobs:
  call-nginx-org-deploy:
    uses: nginxinc/docs-actions/.github/workflows/nginx.org-make-aws.yml@main
    with:
      deployment_env: ${{inputs.deployment_env || 'staging'}}
    secrets:
      AWS_ACCOUNT_ID: ${{secrets.AWS_ACCOUNT_ID}}
      AWS_ROLE_NAME: ${{secrets.AWS_ROLE_NAME}}
      ALLOWED_USERS: ${{secrets.ALLOWED_USERS}}
```
