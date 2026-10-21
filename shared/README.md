# Shared pipeline units

Building blocks for pipelines you write yourself, for example in the pipeline templates repository of your
project (see the [root README](../README.md#using-your-own-pipelines)). The console never copies this folder
into a repository: it only reads `<type>/<compiler>/<provider>/`.

Every unit ends with the one call MFE Orchestrator needs from a service pipeline: registering the pushed
image as a version. What runs before it — tests, scans, signing, the registry — is up to you.

Pin a tag or a commit SHA instead of `main` once you depend on these.

## GitHub Actions

Whole release — build, push, register — as a reusable workflow:

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ['*']

permissions:
  contents: read
  packages: write

jobs:
  release:
    uses: mfe-orchestrator/template-pipelines/.github/workflows/service-docker-release.yml@main
    with:
      service-slug: '%serviceSlug%'
      orchestrator-domain: ${{ vars.MICROFRONTEND_ORCHESTRATOR_DOMAIN || '%domain%' }}
      platforms: linux/amd64,linux/arm64
    secrets:
      api-key: ${{ secrets.MICROFRONTEND_ORCHESTRATOR_API_KEY }}
```

Inputs: `registry` (default `ghcr.io`), `image` (default `ghcr.io/<owner>/<repo>`), `version` (default the
git ref), `context`, `dockerfile`, `platforms`, `build-args`. Secrets `registry-username` and
`registry-password` default to `github.actor` and `GITHUB_TOKEN`. Outputs: `image`, `digest`.

Registration only, after a build of your own:

```yaml
      - uses: mfe-orchestrator/template-pipelines/shared/github/register-service-version@main
        with:
          service-slug: '%serviceSlug%'
          orchestrator-domain: '%domain%'
          api-key: ${{ secrets.MICROFRONTEND_ORCHESTRATOR_API_KEY }}
          image: ghcr.io/acme/orders-be:${{ github.ref_name }}
          digest: ${{ steps.build.outputs.digest }}
```

## GitLab CI

```yaml
# .gitlab-ci.yml
include:
  - remote: https://raw.githubusercontent.com/mfe-orchestrator/template-pipelines/main/shared/gitlab/service-docker.yml

variables:
  MFE_SERVICE_SLUG: '%serviceSlug%'
  MICROFRONTEND_ORCHESTRATOR_DOMAIN: '%domain%'

stages: [test, release]

test:
  stage: test
  script: [./run-tests.sh]

release:
  stage: release
  extends: .mfe-service-docker-release
  needs: [test]
```

`.mfe-service-docker-release` builds `MFE_DOCKERFILE` in `MFE_DOCKER_CONTEXT`, pushes to the GitLab registry
of the project and registers the version, on tags only. To push elsewhere, write your own release job and end
it with the registration:

```yaml
release:
  script:
    - ./build-and-push.sh   # sets MFE_IMAGE, and MFE_IMAGE_DIGEST if you have it
    - !reference [.mfe-register-service-version, script]
```

A self-managed GitLab must be able to reach `raw.githubusercontent.com`; otherwise copy the file into a
project of your instance and include it with `include: project:`.

## Azure Pipelines

Azure reads templates from GitHub through a GitHub service connection, even for a public repository.

```yaml
# azure-pipelines.yml
trigger:
  tags:
    include: ['*']

resources:
  repositories:
    - repository: mfe
      type: github
      name: mfe-orchestrator/template-pipelines
      endpoint: my-github-connection   # GitHub service connection
      ref: refs/heads/main

pool:
  vmImage: ubuntu-latest

variables:
  - group: MFE_ORCHESTRATOR_SECRETS

steps:
  - template: shared/azure_dev_ops/service-docker-release.yml@mfe
    parameters:
      serviceSlug: '%serviceSlug%'
      orchestratorDomain: '%domain%'
      containerRegistry: my-registry-connection
      repository: acme/%serviceSlug%
      registryHost: acme.azurecr.io
```

Registration only: `- template: shared/azure_dev_ops/register-service-version.yml@mfe` with `serviceSlug`,
`image`, `version`, `orchestratorDomain` and optionally `digest`.

## The call itself

For any other CI:

```
POST <console>/api/services/by-slug/<serviceSlug>/versions/<version>
api-key: <deploy API key, MANAGER role>
Content-Type: application/json

{ "image": "ghcr.io/acme/orders-be:1.2.0", "digest": "sha256:…", "commitSha": "…" }
```

`image` is required; `digest` and `commitSha` are optional.
