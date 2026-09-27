# Lightpanda Helm Charts

[![Lint and Test Charts](https://github.com/laghoule/lightpanda_browser/actions/workflows/lint-test.yaml/badge.svg)](https://github.com/laghoule/lightpanda_browser/actions/workflows/lint-test.yaml)
[![Release Charts](https://github.com/laghoule/lightpanda_browser/actions/workflows/release.yaml/badge.svg)](https://github.com/laghoule/lightpanda_browser/actions/workflows/release.yaml)

This repository hosts Helm charts, published as a self-hosted Helm chart
repository via GitHub Pages.

## Charts

| Chart | Description |
| --- | --- |
| [`lightpanda`](./charts/lightpanda) | [Lightpanda Browser](https://lightpanda.io) — AI-native headless browser, deployed as a CDP/WebDriver Bidi server |

See each chart's own `README.md` for detailed configuration options.

## Usage

Add this repository to Helm:

```console
helm repo add lightpanda-charts https://laghoule.github.io/lightpanda_browser/
helm repo update
```

Install a chart:

```console
helm install my-lightpanda lightpanda-charts/lightpanda
```

Alternatively, install directly from a local checkout without adding the
repo:

```console
helm install my-lightpanda ./charts/lightpanda
```

## Repository layout

```
.
├── .github/
│   ├── workflows/
│   │   ├── lint-test.yaml   # ct lint/install on pull requests
│   │   └── release.yaml     # chart-releaser on push to main
│   └── dependabot.yml       # keep GitHub Actions versions up to date
├── charts/
│   └── lightpanda/          # the Lightpanda Browser chart
├── ct.yaml                  # chart-testing configuration
└── LICENSE
```

## How releases work

This repo uses [helm/chart-releaser-action](https://github.com/helm/chart-releaser-action):
on every push to `main` that touches `charts/**`, it:

1. Detects any chart whose `version` in `Chart.yaml` was bumped.
2. Packages it (`helm package`) and creates a GitHub Release named
   `<chart>-<version>` with the packaged `.tgz` attached.
3. Updates `index.yaml` on the `gh-pages` branch, which GitHub Pages serves
   as the Helm repository index consumed by `helm repo add`.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for one-time repository setup
(GitHub Pages, workflow permissions), the release process, and local
validation instructions.

## License

Licensed under the [GNU General Public License v3.0](./LICENSE) (GPLv3).
Note this repository (the Helm chart packaging) is licensed independently
from the upstream [lightpanda-io/browser](https://github.com/lightpanda-io/browser)
project, which has its own license.
