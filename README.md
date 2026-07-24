# rocjay1 Doc Hub Portal

This repository serves as the centralized entry point and "Front Door" for my professional workspace on GitHub Pages.

## Features
- **Project Index:** A curated dashboard of my engineering projects, automated pipelines, and documentation.
- **Consistent Branding:** Synchronized design system (typography, color palette, and layout) across all sub-projects.
- **MkDocs Powered:** Built with the high-performance [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme.
- **Automated GitOps:** GitHub Actions automatically builds and deploys the portal to `rocjay1.github.io` on every push.

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

## Portal URL
- **Live Portal:** [https://docs.roccosmodernsite.net/](https://docs.roccosmodernsite.net/) (configured with verified custom domain)
