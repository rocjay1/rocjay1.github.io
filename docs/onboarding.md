# Onboarding New Project Documentation

This guide outlines the standards and configuration required to onboard a new repository into the **rocjay1 Documentation Hub** with full stylistic consistency.

---

## 1. Design Tokens (Brand Identity)

To maintain a unified look, all sub-projects must use the following design tokens in their `mkdocs.yml`:

### Color & Typography
```yaml
theme:
  name: material
  palette:
    primary: emerald  # Brand color (#10b981)
    accent: emerald
  font:
    text: Segoe UI    # Resume-consistent typography
    code: Roboto Mono
```

### Emerald Sync (Extra CSS)
Add this to a file at `docs/stylesheets/extra.css` in your new project to match the hub's specific header and accent styles:

```css
:root {
  --md-primary-fg-color: #10b981;
  --md-primary-fg-color--light: #34d399;
  --md-primary-fg-color--dark: #059669;
  --md-typeset-a-color: #10b981;
}

.md-typeset h1, .md-typeset h2 {
  color: #111827;
  font-weight: 700;
}
```

---

## 2. Essential Extensions

The following extensions are required for proper rendering of icons, grid cards, and syntax highlighting:

```yaml
markdown_extensions:
  - attr_list
  - md_in_html
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences
  - pymdownx.emoji:
      emoji_index: !!python/name:material.extensions.emoji.twemoji
      emoji_generator: !!python/name:material.extensions.emoji.to_svg
```

---

## 3. Subpath Deployment Configuration

When deploying a standalone repository as a sub-directory of the Hub, you must set the `site_url` correctly.

### mkdocs.yml
```yaml
site_url: https://docs.roccosmodernsite.net/your-repo-name/
```

### GitHub Actions Deployment
Use the modern artifact-based deployment. Ensure your workflow includes the following job:

```yaml
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

---

## 4. Registering with the Hub

Once your new project is live at `docs.roccosmodernsite.net/your-repo-name/`, add a new entry to the **Portfolio Projects** grid in `rocjay1.github.io/docs/index.md`:

```markdown
-   :material-folder-zip: __New Project Name__

    ---

    A brief technical description of the project.

    [:octicons-arrow-right-24: View Docs](https://docs.roccosmodernsite.net/your-repo-name/)
```
