# docs-actions

This repo contains actions for building and deploying, hugo and sphinx based documentation websites for NGINX.

The action pushes to an Azure Storage Container, and assumed an existing reverse proxy (and or CDN) to make the sites public.

By default, it will also purge the provided CDN path.

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
