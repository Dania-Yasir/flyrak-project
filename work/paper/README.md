# Research paper website

This folder contains the static web version of the final capstone research paper.

## Files

- `index.html` — the published paper page
- `styles.css` — responsive academic/portfolio styling
- `script.js` — reading progress and active table-of-contents state

The canonical manuscript remains at `work/research_paper.md`. Final figures remain at `work/figures/` and are copied into the Pages artifact by `.github/workflows/paper-pages.yml` during deployment.

## Local preview

From the repository root:

```bash
python -m http.server 8000 --directory work/paper
```

For a fully local preview with figures, either open the site after deployment or temporarily copy the PNG files from `work/figures/` into `work/paper/assets/`.

## Deployment

GitHub Pages should use **GitHub Actions** as its publishing source. The workflow deploys only the research-paper site, not the full repository.
