# docs-actions

This repo contains actions for building and deploying, hugo and sphinx based documentation websites for NGINX.

The action pushes to an Azure Storage Container, and assumed an existing reverse proxy (and or CDN) to make the sites public.

By default, it will also purge the provided CDN path.

## Usage

### Prerequisites
`AZURE_CREDENTIALS` need to contain service principal credentials with the following roles assigned;
- Storage Blob Data Contributor
- Key Vault Secrets User
- CDN Endpoint Contributor

`AZURE_KEY_VAULT` should contain the name of a key vault, which contains the following secrets;
- productionHostname
- previewHostname
- resourceGroupName
- cdnProfileName
- cdnName
- accountName

The secrets in AKV allow variables to be shared by all repos contributing to the same base doc sites (as a single source of truth), 
without needing to duplicate github secrets or variables across different github organisations.

Built files are pushed to a location in Azure Container Storage based on the repository name and deploy type.

For example;

A repo called `nginxinc/upgraded-octo-umbrella`;
- Preview pushed to `$web/nginxinc/upgraded-octo-umbrella/preview/${prNumber}`
- Production build to `$web/nginxinc/upgraded-octo-umbrella/latest`


### Caller example

Full usage examples;
``` yml
on:
  push:
    branches:
      - main # Set a branch to deploy
    paths:
      - docsDirectory/**
  pull_request:
    branches:
      - "*"
    paths:
      - docsDirectory/**

jobs:
  setPrMerged:
    runs-on: ubuntu-latest
    outputs:
      prMerged: ${{steps.setPrMerged.outputs.prMerged}}
    steps:
      - name: Set PR merged value
        id: setPrMerged
        run: echo "prMerged=${{github.event_name == 'push' && github.ref == 'refs/heads/main'}}" >> $GITHUB_OUTPUT

  call-docs-build-push:
    needs: setPrMerged
    uses: nginxinc/docs-actions/.github/workflows/docs-build-push.yml@main
    with:
      event_action: ${{github.event.action}}
      pr_merged: ${{ needs.setPrMerged.outputs.prMerged}}
      production_url_path: "/"
      preview_url_path: "/previews/docs/"
      docs_source_path: "./public/"
      cdn_content_path: "/*"
      doc_type: "hugo"
    secrets:
      AZURE_CREDENTIALS: ${{secrets.AZURE_CREDENTIALS}}
      AZURE_KEY_VAULT: ${{secrets.AZURE_KEY_VAULT}}
```

Each docs repo has slightly different requirements in terms of inputs, primarily
```yml
event_action: ${{github.event.action}}
pr_merged: ${{ needs.setPrMerged.outputs.prMerged}}
production_url_path: "/rootOfProduct"  (i.e. / for docs , /nginx-gateway-fabric for gateway fabric)
preview_url_path: "/previews/nameOfProduction/"
docs_source_path: "./outputDirOfBuiltDocs/"
cdn_content_path: "/rootOfProduct" or /* for docs
doc_type: "hugo" (hugo for everything except unit)

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
