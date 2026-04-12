---
title: Seed Pipeline — tartalom szinkronizáció site.ts → WordPress
status: draft
type: concept
phase_introduced: "8.5"
canonical: true
repos: [sp-infra, sp-benettcar, sp-exotica, sp-docs]
tags: [seed, pipeline, parity, acf, wordpress, content-sync]
last_updated: 2026-04-12
---

# Seed Pipeline — tartalom szinkronizáció site.ts → WordPress

## Summary (Simple)

A Seed Pipeline az a mechanizmus, amellyel a kliens-oldali kanonikus tartalom (`site.ts`) eljut a WordPress adatbázisba. A tartalom egyetlen forrásból (site.ts) indul, átmegy egy export → import → dump → verify → endpoint smoke láncon, és a végén garantáltan azonos adat van a WP-ben, mint a site.ts-ben. Ez biztosítja, hogy a headless CMS frontend és a WP admin felület ugyanazt az adatot tükrözze.

## Technical Explanation

### Pipeline lépések

```
site.ts (SiteData)
  → [1] export-seed.ts → seed.json
    → [2] import-seed.php → WordPress DB (ACF fields)
      → [3] dump-acf.php → wp-state.json
        → [4] verify-parity.ts → PASS/FAIL (55/55)
          → [5] verify-endpoint.php → endpoint smoke (18/18)
```

| Lépés | Script | Repo | Input | Output |
|-------|--------|------|-------|--------|
| 1. Export | `export-seed.ts` | sp-benettcar | site.ts + mapping.ts | seed.json |
| 2. Import | `import-seed.php` | sp-infra | seed.json | WP DB (ACF fields) |
| 3. Dump | `dump-acf.php` | sp-infra | WP DB + seed.json (kulcsok) | wp-state.json |
| 4. Verify | `verify-parity.ts` | sp-infra | seed.json + wp-state.json | PASS/FAIL |
| 5. Smoke | `verify-endpoint.php` | sp-infra | REST endpoint belső dispatch | 18 image URL ellenőrzés |

### Orchestrátor

`seed-pipeline.ps1` futtatja az összes lépést egy parancsból:

```powershell
.\seed-pipeline.ps1              # teljes pipeline
.\seed-pipeline.ps1 -DryRun     # WP-be nem ír
.\seed-pipeline.ps1 -SkipExport # meglévő seed.json-t használja
```

### seed.json shape

Flat struktúra, három top-level kulcs:

```json
{
  "post_id": "front_page",
  "site_options": {
    "blogname": "...",
    "blogdescription": "..."
  },
  "fields": {
    "bc_hero_title": "...",
    "bc_brand_brands": [{ "name": "...", "logo": "..." }],
    "bc_service_contact": { "title": "...", "phone": "..." }
  }
}
```

- `post_id: "front_page"` → az importer `get_option('page_on_front')`-tal oldja fel
- `fields` lapos: kulcs = ACF field name, érték = közvetlenül ACF-be írandó
- Kind detection automatikus (`detect_kind()`): scalar, repeater, group, image
- Generált fájl — **nem commitolható** (.gitignore)

### Parity definíció

| Mező típus | Egyezési szint | Megjegyzés |
|---|---|---|
| Szöveg (title, description) | Exact match | Karakter-szintű |
| CTA (text, href) | Exact match | |
| Média (image) | Structural match | URL eltérhet, alt egyeznie kell |
| Group sub-field | Exact match | Kulcs-normalizálás szükséges (lásd: p8.5-group-key-normalization.md) |
| Navigation | Out of scope | Config-driven, nem DB-ből |

### Repo boundary

| Fájl | Repo | Indoklás |
|---|---|---|
| `infra/seed/export-seed.ts` | sp-benettcar | Kliens site.ts-t importálja, kliens mapping-et használja |
| `infra/seed/mapping.ts` | sp-benettcar | ACF field nevek kliens-specifikusak |
| `seed/import-seed.php` | sp-infra | ACF API = platform művelet |
| `seed/dump-acf.php` | sp-infra | State dump = platform művelet |
| `seed/verify-parity.ts` | sp-infra | Parity check = platform-szintű gate |
| `seed/seed-pipeline.ps1` | sp-infra | Orchestrátor = platform |

**Miért nem sp-infra az export?** Az export input (site.ts), a mapping, és az export logika mind kliens-specifikus. Sp-infra-ba helyezés cross-repo importot és kliensfüggő kódot vinne az infra rétegbe.

## Why It Exists

Phase 8.3 bevezette az adapter switch-et (JSON → WP), de a WP adatbázisban dummy tartalom volt (Phase 6.4-ből). A site.ts valós, éles tartalmat tartalmazott (~55 mező, 10 szekció). Semmi nem garantálta, hogy a kettő egyezik, és nem volt gate, ami kikényszerítette volna.

A Seed Pipeline ezt az űrt tölti be: automatizált, ellenőrizhető, idempotens tartalom-szinkronizáció.

## Implementation Notes

- **import-seed.php**: `detect_kind()` alapján dönt a write stratégiáról. Scalar → `update_field()`, repeater → `update_field()` tömbbel, group → `update_field()` asszociatív tömbbel, image → `update_field()` URL stringgel + `sideload_local_file()` ha lokális asset.
- **dump-acf.php**: A seed.json kulcsait iterálja, `get_field()` + `get_option()` hívásokkal olvassa a WP-t. Group sub-field-eknél seed-shape kulcs-remapping (suffix match).
- **verify-parity.ts**: Összehasonlít seed.json vs wp-state.json. Image mezőknél structural match (alt egyezés, URL eltérhet). Exit 0 = PASS, 1 = FAIL.
- **verify-endpoint.php**: Belső REST dispatch (`/spektra/v1/site`), 18 image field URL-t ellenőriz (valid URL-e).
- **seed-pipeline.ps1**: PHP 8.4.x auto-select (8.5+ breaks WP-CLI phar). WP-CLI phar auto-download. Dual-form arg parsing (WP-CLI intercepts `--` prefixed custom args).
- **asset-loader.mjs**: Stub loader site.ts ES importjaihoz (brand logók) — projekt-relatív path-ot ad vissza.

## Boundaries and Guardrails

- **Owner**: sp-infra (import, dump, verify, pipeline), sp-benettcar (export, mapping)
- **NEM szabad**: kliens-specifikus logikát sp-infra-ba tenni (mapping, site.ts import)
- **NEM szabad**: seed.json-t commitolni (generált artifact)
- **NEM szabad**: production adapter switch-et engedélyezni PASS nélkül (CMS Runtime Parity Gate)
- **Navigation kívül esik** a parity scope-on — config-driven, nem ACF

## Related Concepts

- [p8.5-parity-gate.md](p8.5-parity-gate.md) — CMS Runtime Parity Gate guardrail
- [p8.5-group-key-normalization.md](p8.5-group-key-normalization.md) — ACF group sub-field kulcs-normalizálás
- [p8.5-image-sideload.md](p8.5-image-sideload.md) — Image sideloading a pipeline-ban
- `sp-docs/content-parity-bootstrap.md` — teljes workstream definíció (ACF mapping táblák, CLI interface-ek)
- `sp-docs/postmortem-p8.5-content-parity.md` — postmortem és sequencing hibák

## Open Questions / TODO

- `seed-pipeline.ps1` jelenleg Windows-only (PowerShell). Cross-platform (bash/CI) adaptáció P11+ scope.
- Automatikus ACF field group sync (site.ts → WP admin field definition) — jövőbeli scope.
- Pre-flight URL validation beépítése a pipeline-ba (lásd: seed-pipeline-blockers.md B2/B5)

## Multi-Client Validation

A pipeline az sp-exotica (2. kliens) seed futtatásával igazoltan generikus:
- sp-infra scriptek (import, dump, verify, endpoint) változtatás nélkül működtek
- Csak a kliens-specifikus `mapping.ts` és `site.ts` szükséges a futtatáshoz
- Eredmény: 39/39 parity, 20/20 endpoint (3 pipeline futtatás a PASS-ig — lásd: [../troubleshooting/seed-pipeline-blockers.md](../troubleshooting/seed-pipeline-blockers.md))
- A generikus fájlok (export-seed.ts, asset-loader.mjs, register-asset-hooks.mjs) azonosnak bizonyultak — elevation jelöltek (lásd: [platform-elevation.md](platform-elevation.md))
