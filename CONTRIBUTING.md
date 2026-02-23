# Contributing to Everest Helm Charts

Thank you for contributing! This guide walks you through making changes to the chart and publishing a release.

## Prerequisites

The following tools must be available in your `PATH`:

| Tool | Purpose |
|---|---|
| [`helm`](https://helm.sh/docs/intro/install/) ≥ 3.x | Packaging and dependency management |
| [`yq`](https://github.com/mikefarah/yq) ≥ 4.x | YAML patching in `prepare-chart` |
| [`docker`](https://docs.docker.com/get-docker/) | Running `helm-docs` for doc generation |
| [`git`](https://git-scm.com/) | Worktree-based `gh-pages` publishing |

All `make` targets must be run from the `charts/everest/` directory:

```bash
cd charts/everest
```

---

## Repository Layout

```
helm-charts/
└── charts/
    └── everest/               ← main chart (work here)
        ├── Makefile           ← all automation lives here
        ├── Chart.yaml         ← chart version & dependencies
        ├── values.yaml        ← default values
        ├── templates/         ← Kubernetes manifest templates
        └── charts/
            ├── everest-crds/  ← CRD sub-chart
            ├── everest-db-namespace/
            ├── common/
            └── pmm3/
```

The `gh-pages` branch serves the public Helm repository at:
```
https://openeverest.github.io/helm-charts/
```

---

## Making Changes

### 1. Fork & clone

```bash
git clone https://github.com/openeverest/helm-charts.git
cd helm-charts
git checkout -b my-feature
```

### 2. Install dependencies

```bash
cd charts/everest
make deps
```

This adds the required Helm repos (`prometheus-community`, `victoriametrics`, `percona`, `percona-olm`) and runs `helm dependency update` for all sub-charts.

### 3. Edit the chart

Make your changes under `charts/everest/templates/`, `values.yaml`, or the relevant sub-chart.

### 4. Prepare your PR

Before opening a pull request, run:

```bash
make prepare-pr
```

This will:
- Re-run `helm dependency update` for all charts
- Regenerate `README.md` from `README.md.gotmpl` via `helm-docs`
- Optionally regenerate CRDs from upstream (see [Updating CRDs](#updating-crds) below)

Commit the resulting changes along with your feature work.

---

## Updating CRDs

If your change requires refreshing the CRDs from the upstream operator repository, run:

```bash
make crds-gen CRD_VERSION=<tag-or-branch>
make link-crds
```

- `CRD_VERSION` defaults to `main`. Use a specific tag (e.g. `v1.14.0`) to pin to a release.
- `crds-gen` fetches CRDs from [`openeverest/openeverest-operator`](https://github.com/openeverest/openeverest-operator) via `kustomize` (runs in Docker — no local `kustomize` install needed).
- `link-crds` updates the symlinks in the top-level `crds/` directory.

Commit the updated files in `charts/everest-crds/templates/` and `crds/`.

---

## Publishing a Release

> **Note:** Only maintainers with push access to `gh-pages` should do this step.

### Full release (bump version → publish)

```bash
make release-and-publish VERSION=1.2.3
```

This single command:
1. Stamps `version` and `appVersion` in all three `Chart.yaml` files (`everest`, `everest-crds`, `everest-db-namespace`)
2. Updates image references in `values.yaml`
3. Re-pins dependency versions in `Chart.yaml`
4. Runs `helm dependency update`
5. Packages the chart into a `.tgz`
6. Regenerates `index.yaml` on the `gh-pages` branch (preserving prior releases)
7. Commits and pushes `gh-pages`

After pushing, the chart is available within minutes:

```bash
helm repo add openeverest https://openeverest.github.io/helm-charts/
helm repo update
helm search repo openeverest/everest
```

### Step-by-step (if you need to separate the release from the publish)

```bash
# 1. Bump versions and update dependencies
make release VERSION=1.2.3

# 2. Commit the version bump to main
git add -A
git commit -m "Release v1.2.3"
git push origin main

# 3. Publish packaged chart to gh-pages
make publish VERSION=1.2.3
```

### Dev / pre-release

For a dev release using `-dev` image variants and telemetry disabled:

```bash
make release-dev VERSION=1.2.3-dev
```

This does **not** call `publish` — dev builds are not pushed to the public Helm index.

---

## Available `make` Targets (summary)

| Target | Description |
|---|---|
| `make deps` | Add Helm repos and update all dependencies |
| `make prepare-pr` | Sync deps, regenerate docs, refresh CRDs — run before opening a PR |
| `make release VERSION=x.y.z` | Bump all versions and update images/deps |
| `make publish VERSION=x.y.z` | Package chart and push to `gh-pages` |
| `make release-and-publish VERSION=x.y.z` | `release` + `publish` in one step |
| `make release-dev VERSION=x.y.z` | Dev release with `-dev` images, telemetry off |
| `make crds-gen CRD_VERSION=<ref>` | Regenerate CRD templates from upstream operator |
| `make link-crds` | Refresh symlinks in `crds/` from CRD templates |
| `make docs-gen` | Regenerate `README.md` via `helm-docs` (requires Docker) |
