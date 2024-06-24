# docs-actions

This repo contains actions for building and deploying, hugo and sphinx based documentation websites for NGINX.

The action pushes to an Azure Storage Container, and assumed an existing reverse proxy (and or CDN) to make the sites public.

By default, it will also purge the provided CDN path.

## Usage

### Security considerations
`AZURE_CREDENTIALS` need to contain service principal credentials with the following roles assigned;
- **Storage Blob Data Contributor**
- **Key Vault Secrets User**
- **CDN Endpoint Contributor**

The service principal (or managed identity) references in the AZURE_CREDENTIALS should have _only_ the roles
listed above. Assigning any more roles is a potential security risk.

`AZURE_KEY_VAULT` should contain the name of a key vault, which contains the following secrets;
- productionHostname
- previewHostname
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

A repo called `nginxinc/upgraded-octo-umbrella`;
- Preview pushed to `$web/nginxinc/upgraded-octo-umbrella/preview/${prNumber}`
- Production build to `$web/nginxinc/upgraded-octo-umbrella/latest`



### Caller example

Place this workflow action in the caller repository (docs, kubernetes-ingress, etc.)

Full usage example taken from the [octo-umbrella](https://github.com/nginxinc/upgraded-octo-umbrella/) test repository:
``` yml
name: Build and deploy (octo-umbrella)
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
      production_url_path: "/octo-umbrella/"
      preview_url_path: "/previews/octo-umbrella/"
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
