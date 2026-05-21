# Fliphetic

Fliphetic is a deployment and management system for a virtual pinball cabinet.
It runs three Chromium kiosk screens and one or more ESP32 boards, and lets you
register student "apps" from git, then load any of them onto the cabinet from a
single web dashboard.

This repository hosts the **documentation site**, published with GitHub Pages:

**https://pandormedia.github.io/fliphetic/**

The documentation is bilingual (English and French) and covers the full
deployment: hardware and software requirements, cabinet provisioning, installing
Fliphetic, configuration, day to day operations, building apps, ESP32 firmware,
architecture, and troubleshooting.

## Editing the docs

The site is built with [MkDocs](https://www.mkdocs.org/) and the
[Material](https://squidfunk.github.io/mkdocs-material/) theme, with the
`mkdocs-static-i18n` plugin for the English and French versions.

```sh
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve        # live preview at http://127.0.0.1:8000
mkdocs build        # render the static site into site/
```

Pages live in `docs/`. English files have no language suffix (`install.md`);
French files use the `.fr` suffix (`install.fr.md`).

Every push to `main` rebuilds and redeploys the site through the GitHub Actions
workflow in `.github/workflows/docs.yml`.
