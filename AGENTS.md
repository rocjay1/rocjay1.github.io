# AGENTS.md

This repository publishes the `rocjay1` documentation portal through GitHub
Pages.

## Repository structure

- `docs/` contains authored documentation and blog posts.
- `docs/engineering/` contains cross-repository architecture and standards.
- `mkdocs.yml` owns navigation, theme, plugins, and Markdown extensions.
- `pyproject.toml` and `uv.lock` own the reproducible documentation toolchain.
- `site/` is generated output and must not be committed.

## Validation

```bash
uv sync --locked
uv run mkdocs build --strict
```

Keep navigation and repository-relative links current when moving or adding
documents. Pin third-party GitHub Actions to immutable commit SHAs, use
Conventional Commits, and do not deploy or change GitHub Pages settings unless
explicitly requested.
