---
title: Image Sideloading a Seed Pipeline-ban
status: draft
type: implementation
phase_introduced: "8.5"
canonical: true
repos: [sp-infra, sp-benettcar]
tags: [image, sideload, media-library, acf, pipeline, wordpress]
last_updated: 2026-04-11
---

# Image Sideloading a Seed Pipeline-ban

## Summary (Simple)

A seed pipeline-ban a képek kétféle forrásból érkezhetnek: external URL-ek (pl. Unsplash) és lokális asset fájlok (pl. brand logók a kliens repóban). Mindkét esetben a cél: a kép bekerüljön a WordPress media library-be, és a REST endpoint valid WP media URL-t adjon vissza. Az `import-seed.php` automatikusan felismeri a képreferenciákat és sideload-olja őket a WP-be.

## Technical Explanation

### Image flow a pipeline-ban

```
site.ts
  ├── external URL (pl. "https://images.unsplash.com/...")
  │     → seed.json: { url: "https://...", alt: "..." }
  │       → import-seed.php: sideload_external_image() → WP attachment
  │
  └── ES import (pl. import vwLogo from './assets/vw-logo.jpg')
        → asset-loader.mjs stub → "src/assets/brands/vw-logo.jpg"
          → seed.json: repeater row { logo: "src/assets/brands/vw-logo.jpg" }
            → import-seed.php: looks_like_image_ref() → sideload_local_file() → WP attachment
```

### Két sideload mechanizmus

**1. External URL sideload** (`sideload_external_image()`):
- Felismeri a `{ url, alt }` shape-et, ahol az URL http(s)-sel kezdődik
- `media_sideload_image()` WP API-val letölti és WP attachment-ként tárolja
- Attachment ID-t kap vissza → `wp_get_attachment_image_src()` → WP media URL
- Érintett mezők: bc-hero background, bc-about image, bc-team member images, bc-gallery images

**2. Lokális fájl sideload** (`sideload_local_file()`):
- `looks_like_image_ref()` heurisztika: string értékek, amelyek képfájl-kiterjesztésre végződnek (.jpg, .png, .svg stb.) és nem URL-ek
- A fájl path-ot a WordPress theme/plugin gyökeréhez viszonyítva keresi, vagy abszolút path-ként
- `wp_insert_attachment()` + `wp_generate_attachment_metadata()` — lokális fájlból WP attachment
- **Idempotens**: ha a fájlnév alapján már létezik attachment, CACHED (nem duplikál)
- Érintett mezők: 6 brand logó (`bc_brand_brands` repeater row `logo` mező)

### Asset Loader (kliens oldal)

A `site.ts` ES importokat használ a brand logókhoz:
```typescript
import vwLogo from '../assets/brands/vw-logo.jpg';
```

Vite build/dev módban ez asset URL-lé oldódik. A seed export kontextusban (tsx + Node) nincs Vite, ezért az `asset-loader.mjs` stub loader fut, ami projekt-relatív path-ot ad vissza.

**Fontos**: az asset-loader Windows abszolút path-okat adott az első verzióban (`/D:/Projects/...`). Fix: `PROJECT_ROOT` számítás a loader pozíciójából, és relatív path generálás.

### Endpoint oldalon

A REST endpoint kétféle image formátumot kezel:

- **Image field (object shape)**: `spektra_normalize_media($field)` — WP attachment array-t vagy attachment ID-t normalizál `{ url, alt, width, height }` shape-re
- **Plain URL field (string)**: `spektra_resolve_image_url($field)` — attachment ID-t, ACF array-t, vagy URL string-et egyaránt plain URL-lé old

### Számok

| Forrás | Darab | Példa |
|--------|-------|-------|
| External URL | 12 | hero background, about image, 3 team member, 7 gallery |
| Lokális fájl | 6 | brand logók (VW, Audi, Seat, Skoda, Cupra, Porsche) |
| **Összesen** | **18** | Endpoint smoke: 18/18 PASS |

## Why It Exists

A site.ts image referenciák nem közvetlenül alkalmazhatók WP-ben:
- External URL-ek: a WP media library nem ismeri, nincs attachment metadata
- Lokális fájlok: ES import path-ok, amelyek csak Vite kontextusban működnek

Sideloading nélkül a REST endpoint:
- Attachment ID-kat (integer) adna vissza URL helyett
- Vagy `null`-t, ha a mező nincs a media library-ben
- A frontend törött `<img>` tag-eket renderelne

## Implementation Notes

- **import-seed.php** (sp-infra/seed/):
  - `detect_kind()`: image shape detection `{ url, alt }` pattern alapján
  - `looks_like_image_ref()`: bare string image detection (kiterjesztés heurisztika)
  - `sideload_local_file()`: idempotens, fájlnév-alapú duplikáció-védelem
  - `sideload_external_image()`: `media_sideload_image()` wrapper
- **media-helper.php** (sp-infra/acf/):
  - `spektra_normalize_media()`: attachment ID/array → `{ url, alt, width, height }`
  - `spektra_resolve_image_url()`: bármilyen image ref → plain URL string
- **sections.php** (sp-infra/acf/):
  - bc-brand: `spektra_resolve_image_url($row['logo'])` — plain URL a repeater row-ból
  - bc-gallery: `spektra_resolve_image_url($row['src'])` — plain URL
  - bc-team: `spektra_normalize_media($row['image'])` — full media shape
  - bc-hero, bc-about: `spektra_normalize_media(get_field(...))` — full media shape
- **asset-loader.mjs** (sp-benettcar/infra/seed/): stub loader, projekt-relatív path output
- **Commit-ek**: `7aa1964` (initial), `b10b5cb` (sideload + footer nav), `6914cb6` (resolve + smoke)

## Boundaries and Guardrails

- **Owner**: sp-infra (import, media-helper, sections), sp-benettcar (asset-loader, export)
- **Az importer soha nem módosítja a site.ts-t** — egyirányú flow (site.ts → WP)
- **A sideload idempotens** — újrafuttatáskor CACHED, nem duplikálja az attachment-eket
- **NEM szabad**: a media-helper-ben kliens-specifikus field name-eket hardcode-olni — generic utility marad
- **NEM szabad**: az asset-loader-ben Vite-függő logikát használni — Node-kompatibilis stub kell

## Related Concepts

- [p8.5-seed-pipeline.md](p8.5-seed-pipeline.md) — a pipeline, amelynek a 2. lépése (import) végzi a sideload-ot
- [p8.5-parity-gate.md](p8.5-parity-gate.md) — az endpoint smoke test, amely a sideload eredményét ellenőrzi
- `sp-docs/content-parity-bootstrap.md` §6.5 — eredeti image handling spec

## Open Questions / TODO

- SVG fájlok: a WP alapértelmezetten nem engedi SVG upload-ot. Jelenleg a brand logók JPG/PNG-k. Ha SVG logók kellenek, WP SVG support plugin vagy custom filter szükséges.
- Image optimalizálás (WebP konverzió, srcset generálás) — a WP media library alapból generál szeleteket, de a headless frontend nem feltétlenül használja ezeket.
