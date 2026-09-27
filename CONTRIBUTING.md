# Contributing

This document is for maintainers and contributors of this Helm chart
repository — not for end users of the `lightpanda` chart (see the main
[README.md](./README.md) and [charts/lightpanda/README.md](./charts/lightpanda/README.md)
for that).

## One-time repository setup

These steps must be performed once, in the GitHub repository settings, by
whoever administers this repo (they can't be done from chart files alone):

1. **Push this repository to GitHub**, with `main` as the default branch.
2. **Workflow permissions:** in *Settings → Actions → General → Workflow
   permissions*, select **"Read and write permissions"**. The release
   workflow needs this to push to the `gh-pages` branch and create releases
   using the default `GITHUB_TOKEN`.
3. **First release:** merge a change to `main` (e.g. the initial commit) so
   the release workflow runs once — this creates the `gh-pages` branch.
4. **Enable GitHub Pages:** in *Settings → Pages*, set **Source** to
   **"Deploy from a branch"**, branch **`gh-pages`**, folder **`/ (root)`**.
5. The chart repository is then served at
   `https://laghoule.github.io/lightpanda_browser/`.

## Cutting a new release

1. Bump `version` (and `appVersion`, if the underlying app changed) in
   `charts/lightpanda/Chart.yaml`.
2. Open a PR — CI (`lint-test.yaml`) will run `ct lint`/`ct install` against
   it automatically.
3. Merge to `main` — `release.yaml` then packages the chart, creates a
   GitHub Release, and updates `index.yaml` on `gh-pages`.

## Local validation

```console
# Lint
helm lint charts/lightpanda

# Render templates
helm template test charts/lightpanda
```

The CI workflow also runs [chart-testing](https://github.com/helm/chart-testing)
(`ct`), which does more than plain `helm lint`/`helm template`: it validates
`Chart.yaml` conventions (version bump, maintainers), lints the rendered
YAML, and actually installs the chart on a real (ephemeral `kind`) cluster
to catch issues `helm template` can't see. To reproduce this locally,
without installing `ct` yourself, use its official Docker image:

```console
docker run --rm -it -v "$(pwd)":/workdir --workdir /workdir \
  quay.io/helmpack/chart-testing:v3.11.0 \
  ct lint --target-branch main

docker run --rm -it -v "$(pwd)":/workdir --workdir /workdir \
  --network host \
  quay.io/helmpack/chart-testing:v3.11.0 \
  ct install --target-branch main
```

(`ct install` needs a reachable Kubernetes cluster — e.g. point `KUBECONFIG`
at a local `kind`/`minikube` cluster.)
