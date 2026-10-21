# template-pipelines

The pipelines MFE Orchestrator commits into a repository it creates from a template.

## Layout

```
<type>/<compiler>/<provider>/                 # e.g. remote/vite/github/
<type>/<compiler>/<templateSlug>/<provider>/  # optional, for one template only
shared/                                       # building blocks, never copied: see shared/README.md
```

- `type` and `compiler` are those of the template (`host`, `remote`, `service`; `vite`, `webpack`, `docker`…)
- `provider` is `github`, `gitlab` or `azure_dev_ops`
- **GitHub**: the folder becomes `.github/` and replaces the workflows of the template
- **GitLab** / **Azure DevOps**: `.gitlab-ci.yml` / `azure-pipelines.yml` replace the file at the root

A template with no folder here keeps the pipeline it ships with. That is the case of the service templates,
whose `service/docker` pipelines live in their own repositories.

Placeholders replaced when the repository is created: `%microfrontendSlug%` and `%serviceSlug%` (the slug),
`%domain%` (the URL of the console).

## Using your own pipelines

In the console, **Settings → Pipeline templates**, point the project at a repository of yours with the same
layout, at a branch or tag. Repositories created from a template afterwards get your pipelines instead of these;
a template your repository has no folder for keeps its own, without falling back to this repository.

For example, one release pipeline for every service on GitHub:

```
service/docker/github/workflows/release.yml
```

calling the reusable workflow described in [shared/README.md](shared/README.md).
