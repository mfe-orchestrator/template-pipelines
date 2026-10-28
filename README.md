# template-pipelines

Official CI/CD pipelines of the Flightroom console for frontend repositories. When the console scaffolds a frontend it downloads the `main` branch of this repository and copies `<type>/<compiler>/<provider>` into the new repository.

| Folder level | Values |
| --- | --- |
| `type` | `host`, `remote` |
| `compiler` | `vite`, `webpack`, `single-spa`, `micro-lc`, `webcomponent`, `generic` |
| `provider` | `github` (`workflows/build-and-deploy.yml`, `dependabot.yml`), `gitlab` (`.gitlab-ci.yml`), `azure_dev_ops` (`azure-pipelines.yml`) |

The console replaces three placeholders in the copied files: `%microfrontendSlug%` and `%serviceSlug%` (slug of the frontend), `%domain%` (API base URL of the installation). Everything else is left untouched.

## What a pipeline does

Every pipeline runs on Node 24 with pnpm (from the `packageManager` field of `package.json`) and releases only on a tag; the tag is the version.

- **Remote** (bundle frontend): builds, then uploads the contents of `dist/` to the console (`POST /api/frontends/by-slug/<slug>/upload/<version>`). Done by `mfe-orchestrator-hub/github-action` on GitHub, the `flightroom-upload@1` task on Azure DevOps, `curl` on GitLab. `single-spa` and `micro-lc` remotes also check that the entry point (`dist/index.js`, `dist/parcel.js`) exists.
- **Host** (Docker frontend): builds, builds and pushes an image that serves `dist/` (the template's `Dockerfile`), then registers it on the console, which does not deploy it, it only learns the version:

  ```
  POST ${DOMAIN%/}/api/frontends/by-slug/<slug>/versions/<tag>
  api-key: <FLIGHTROOM_API_KEY>
  Content-Type: application/json

  {"image": "<registry host>/<repository>:<tag>", "digest": "sha256:…", "commitSha": "<sha>"}
  ```

  `image` is required and must be what the cluster pulls; `digest` and `commitSha` are optional (the Azure DevOps pipeline sends no `digest`). The API key needs the `MANAGER` role.

Host build-time variables depend on the bundler: `VITE_MFE_BACKEND_URL`, `VITE_MFE_PROJECT_ID`, `VITE_MFE_ENVIRONMENT` for `vite`, `single-spa`, `micro-lc` and `webcomponent` hosts; `MFE_BACKEND_URL`, `MFE_PROJECT_ID`, `MFE_ENVIRONMENT` for `webpack` hosts (see the `.env.example` of the template). `generic` hosts get none. `*_PROJECT_ID` is required, `*_BACKEND_URL` defaults to `%domain%/api`, `*_ENVIRONMENT` is optional.

## Variables, secrets and connections per provider

`FLIGHTROOM_API_KEY` is the deploy key. The console creates it when it connects the repository provider (GitHub: repository secret; GitLab: group variable; Azure DevOps: variable group `MFE_ORCHESTRATOR_SECRETS`).

### GitHub Actions

| Name | Kind | Needed by | Notes |
| --- | --- | --- | --- |
| `FLIGHTROOM_API_KEY` | secret | all | created by the console |
| `GITHUB_TOKEN` | automatic | hosts | pushes the image to `ghcr.io/<owner>/<repo>`, no setup; the workflow asks for `packages: write` |
| `MICROFRONTEND_ORCHESTRATOR_DOMAIN` | variable, optional | all | console URL, defaults to `%domain%` |
| `VITE_MFE_PROJECT_ID` (`MFE_PROJECT_ID` on webpack) | variable | hosts | required |
| `VITE_MFE_BACKEND_URL`, `VITE_MFE_ENVIRONMENT` (`MFE_…` on webpack) | variable, optional | hosts | |

The workflow can also be started by hand (`workflow_dispatch`) with the version to release. The package must be readable by the cluster (public, or a pull secret).

### GitLab CI

| Name | Kind | Needed by | Notes |
| --- | --- | --- | --- |
| `FLIGHTROOM_API_KEY` | group or project CI/CD variable | all | created by the console; it must not be protected, or tags must be protected too, otherwise tag pipelines do not see it |
| `CI_REGISTRY`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`, `CI_REGISTRY_IMAGE` | predefined | hosts | the image goes to the project's container registry; the Container Registry must be enabled |
| `MICROFRONTEND_ORCHESTRATOR_DOMAIN` | CI/CD variable, optional | all | console URL, defaults to `%domain%` (a CI/CD variable wins over the one in the file) |
| `VITE_MFE_PROJECT_ID` (`MFE_PROJECT_ID` on webpack) | CI/CD variable | hosts | required |
| `VITE_MFE_BACKEND_URL`, `VITE_MFE_ENVIRONMENT` (`MFE_…` on webpack) | CI/CD variable, optional | hosts | |

Hosts need a runner that can run Docker-in-Docker (`docker:29-dind`, privileged).

### Azure DevOps

| Name | Kind | Needed by | Notes |
| --- | --- | --- | --- |
| `MFE_ORCHESTRATOR_SECRETS` | variable group, authorized for the pipeline | all | holds `FLIGHTROOM_API_KEY`, created by the console |
| Flightroom extension (task `flightroom-upload@1`) | extension installed in the organization | remotes | |
| `MFE_ORCHESTRATOR_REGISTRY_CONNECTION` | pipeline variable | hosts | name of the Docker registry service connection |
| `MFE_ORCHESTRATOR_IMAGE_REPOSITORY` | pipeline variable | hosts | repository path in the registry, e.g. `my-org/<slug>` |
| `MFE_ORCHESTRATOR_REGISTRY_HOST` | pipeline variable, optional | hosts | registry host prepended to the image reference sent to the console, e.g. `myregistry.azurecr.io`; set it whenever the connection is not Docker Hub, the cluster needs the full reference |
| `MICROFRONTEND_ORCHESTRATOR_DOMAIN` | pipeline variable, optional | all | console URL, defaults to `%domain%` |
| `VITE_MFE_PROJECT_ID` (`MFE_PROJECT_ID` on webpack) | pipeline variable | hosts | required |
| `VITE_MFE_BACKEND_URL`, `VITE_MFE_ENVIRONMENT` (`MFE_…` on webpack) | pipeline variable, optional | hosts | |

The release stage runs only for tags, so the variable group is loaded only there.

## Dependency updates

On GitHub, `dependabot.yml` opens weekly updates for npm, the GitHub Actions and (hosts) the Docker base image. GitLab and Azure DevOps do not read it: use Renovate there if needed.
