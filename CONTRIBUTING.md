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
   permissions*, select **"Read and write permissions"**. Do this *before*
   the first push/run — the release workflow needs it to push to the
   `gh-pages` branch and create releases using the default `GITHUB_TOKEN`.
3. **First release:** push/merge a change to `main` that touches
   `charts/**` (e.g. the initial commit), or manually trigger the
   **Release Charts** workflow from the *Actions* tab (`workflow_dispatch`).
   This lets `chart-releaser` create the `gh-pages` branch itself (as an
   orphan branch containing `index.yaml` + the packaged `.tgz`) — **do not**
   create `gh-pages` manually from the GitHub UI, or it won't be in the
   format `chart-releaser` expects.
4. **Enable GitHub Pages:** in *Settings → Pages*, set **Source** to
   **"Deploy from a branch"**, branch **`gh-pages`**, folder **`/ (root)`**.
   (This option only appears once `gh-pages` exists, i.e. after step 3 ran
   successfully.)
5. The chart repository is then served at
   `https://laghoule.github.io/lightpanda_browser/`.

### Troubleshooting: no release / empty `index.yaml`

If the **Release Charts** run finishes green but no GitHub Release or tag
appears, and `gh-pages` doesn't contain an `index.yaml`, it's almost always
step 2 above being done *after* the run (permissions were still read-only
when `chart-releaser` tried to push). Fix:

1. Confirm *Settings → Actions → General → Workflow permissions* is set to
   **"Read and write permissions"**.
2. If a `gh-pages` branch already exists but was created manually (e.g. it's
   just a copy of `main` instead of an orphan branch with `index.yaml`),
   delete it (*Settings → Branches*, or `git push origin --delete gh-pages`).
3. Re-run the **Release Charts** workflow from the *Actions* tab (**Run
   workflow**, thanks to its `workflow_dispatch` trigger) — `chart-releaser`
   will recreate `gh-pages` correctly.
4. Re-point *Settings → Pages* at the freshly created `gh-pages` branch.

## Cutting a new release

1. Bump `version` (and `appVersion`, if the underlying app changed) in
   `charts/lightpanda/Chart.yaml`.
2. Open a PR — CI (`lint-test.yaml`) will run `ct lint`/`ct install` against
   it automatically.
3. Merge to `main` — `release.yaml` then packages the chart, creates a
   GitHub Release, and updates `index.yaml` on `gh-pages`.

### Note: pushes that don't bump the version

`release.yaml` triggers on **any** push to `main` touching `charts/**`, not
just version bumps (e.g. a `README.md` or `values.yaml` fix without a
version change). `chart-releaser` is configured with `skip_existing: true`,
so in that case it just detects that a release for the current version
already exists and skips it (no-op, workflow still shows green). If you
see the run fail instead of skipping, check that `skip_existing` is still
set in `release.yaml` — without it, `chart-releaser` errors out instead of
skipping when trying to re-publish an existing version.

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
