# rocjay1.github.io

This repository serves as the centralized entry point and personal blog on GitHub Pages.

## Features

- **Blog Feed:** A clean, minimal, chronologically organized writings space powered by the Material for MkDocs blog plugin.
- **Flexible Navigation:** Structured layout to support resume and project documentation integration in the future.
- **MkDocs Powered:** Built with the high-performance [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.
- **Security Scanning:** Outfitted with automated GitLeaks scanning on every commit and pull request to check for exposed secrets.
- **Dependency Management:** Configured with Dependabot to automatically track and update workflows and package dependencies.
- **Automated GitOps:** GitHub Actions automatically builds and deploys the portal to `docs.roccosmodernsite.net` on every push to `main`.

## Local Development

To preview the portal locally, ensure you have [`uv`](https://docs.astral.sh/uv/) installed:

1. Install dependencies:

   ```bash
   uv sync
   ```

2. Serve the site:

   ```bash
   uv run mkdocs serve
   ```

3. Open `http://127.0.0.1:8000` in your browser.

## CI/CD Pipeline

The deployment is handled by `.github/workflows/deploy.yml`. It orchestrates the Python environment, dependency installation, and site publication to GitHub Pages.

## Engineering Standards

The portal also documents the shared repository scaffolding, documentation,
CI, and deployment conventions used across the `rocjay1` namespace. See
[`docs/engineering/repository-standards.md`](docs/engineering/repository-standards.md).

## Portal URL

- **Live Portal:** [https://docs.roccosmodernsite.net/](https://docs.roccosmodernsite.net/) (configured with verified custom domain)
