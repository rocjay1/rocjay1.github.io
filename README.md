# rocjay1.github.io

This repository serves as the documentation portal for the `rocjay1` repositories, published with GitHub Pages.

## Features

- **MkDocs Powered:** Built with the high-performance [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.
- **Security Scanning:** Outfitted with automated GitLeaks scanning on pull
  requests, `main` pushes, and a weekly full-history schedule.
- **Dependency Management:** Configured with Dependabot to automatically track and update workflows and package dependencies.
- **Automated GitOps:** GitHub Actions automatically builds and deploys the portal to `docs.roccosmodernsite.net` on every push to `main`.

## Local Development

To preview the portal locally, ensure you have [`uv`](https://docs.astral.sh/uv/) installed:

1. Install dependencies:

   ```bash
   uv sync --locked
   ```

2. Serve the site:

   ```bash
   uv run mkdocs serve
   ```

3. Open `http://127.0.0.1:8000` in your browser.

## CI/CD Pipeline

The deployment is handled by `.github/workflows/deploy.yml`. It orchestrates the Python environment, dependency installation, and site publication to GitHub Pages.

## Portal URL

- **Live Portal:** [https://docs.roccosmodernsite.net/](https://docs.roccosmodernsite.net/) (configured with verified custom domain)
