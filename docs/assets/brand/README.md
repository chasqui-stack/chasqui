# Chasqui brand assets

The mark: a **chasqui** — the Inca relay messenger — running with a letter,
hair streaming into a chat bubble. Flat, geometric, Andean-textile cuts.

| File | Use |
|---|---|
| `chasqui-icon.svg` / `chasqui-icon-512.png` | Square mark on charcoal — org/app avatars, favicons, OG thumbnails |
| `chasqui-logo.svg` / `chasqui-logo.png` | Horizontal lockup (icon + wordmark) — README headers, site, docs |
| `chasqui-og.png` | 1280×640 — GitHub social preview (repo Settings → Social preview) |

## Palette (canonical — snap any export to these)

| Token | Hex | Role |
|---|---|---|
| Amber | `#EA9B27` | Primary accent (the sun, the hair) |
| Terracotta | `#C94B22` | Secondary accent / warm emphasis |
| Cream | `#EFDEC4` | Light surfaces, wordmark on dark |
| Charcoal | `#1C1917` | Dark canvas, ink on light |

UI tokens are *derived* from this palette (≈90% neutrals + amber as the
single accent), not copied verbatim — see `admin/DESIGN.md` once the panel
adopts the identity. Amber on white fails WCAG for text: use it as a fill
with dark text on top, or darken it for links.

Wordmark typography: generated lockup approximates a bold geometric
grotesque; for new compositions use **Space Grotesk** (OFL) rather than
redrawing letters.
