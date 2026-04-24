# Portfolio Site — Dev Notes

## Safari SVG Icon Fix
**Problem:** SVG icons in `.ti img` elements render correctly in Chrome locally and remotely, and in Safari locally — but blow up to full size in Safari on GitHub Pages.

**Cause:** Safari is stricter about SVG intrinsic dimensions than Chrome. When an SVG file doesn't have explicit `width`/`height` attributes baked into the SVG markup, Safari ignores CSS `width`/`height` rules and renders at natural/huge size. This only shows up remotely (GitHub Pages CDN) and not locally, likely due to different caching/loading behaviour.

**Fix:** In `assets/style.css`, the `.ti img` rule needs `!important`, `max-width`, and `max-height`:
```css
.ti img { width: 12px !important; height: 12px !important; max-width: 12px; max-height: 12px; flex-shrink: 0; }
```

---

## Git Workflow (GitHub Pages)
The repo is `dallan5/dallan5.github.io`. Files sit at the repo root (not in a subfolder). Standard push flow:

```bash
git add .
git commit -m "message"
git push origin main
```

If push is rejected (remote has changes you don't):
```bash
git pull origin main --rebase
git push origin main
```

If rebase causes conflicts (e.g. binary files), abort and force push your local version:
```bash
git rebase --abort
git push origin main --force
```

Force push is safe here — it's a solo personal site.

---

## Icon Sources
- Tech stack icons: Simple Icons CDN — `https://cdn.simpleicons.org/[slug]/4a4540`
  - Use hex `4a4540` to match `--ink-mid` colour
  - Save to `assets/images/icons/[name].svg`
- VES logo: `https://vesglobal.org/wp-content/uploads/2023/12/VES-Logo-2x.png`
- For dark backgrounds (award callout): `filter: brightness(0) invert(1)` makes the logo white

---

## File Structure
```
portfolio-site/
├── index.html          — main page
├── job-chain.html      — Job Chain deep dive
├── hdri-ml.html        — HDRI from Backplate deep dive
├── ves-award.html      — Queen Harpy / credits page
└── assets/
    ├── style.css
    ├── images/
    │   ├── icons/      — SVG tech icons + VES logos
    │   ├── credits/    — show poster images
    │   ├── ves/        — ceremony photos + creature renders
    │   ├── job-chain/  — documentation diagrams
    │   ├── hdri-backplate/ — ML training outputs
    │   ├── cfx-groom-pipeline/
    │   ├── megascan-library/
    │   └── virtual-production/
    └── video/
        └── training_progress.mp4
```
