---
title: Repeatable Content Source Strategy
status: draft
type: concept
phase_introduced: "12"
canonical: true
repos: [sp-infra, sp-benettcar, sp-exotica]
tags: [acf, cpt, repeater, source-strategy, baseline, governance]
last_updated: 2026-04-18
---

# Repeatable Content Source Strategy

## Summary (Simple)

A Spektra WordPress free baseline nem függhet ACF Pro-tól. A repeatable content (galéria képek, csapattagok, termékek, márkák stb.) forrás stratégián keresztül kezelt. Három strategy létezik: `cpt_collection` (default free baseline), `fixed_slots` (bounded kis tartalom), `acf_repeater_optional` (opcionális jövőbeli Pro source). A frontend SiteData shape mindig stabil marad — a source strategy csere nem érint frontend prop-okat.

## Technical Explanation

### A governance mismatch

DR-003 ("Fixed Field Groups — ACF Free elég") implicit módon ACF Free sufficiency-t állított. A valóságban az implementáció inception óta ACF Repeater mezőket használ mindkét kliensben (sp-benettcar: 8 repeater field / 6 szekció, sp-exotica: 4 repeater field / 4 szekció). Az ACF Repeater ACF Pro feature. Ez nem drift — a governance és az implementáció sosem volt konzisztens.

### Source Strategies

#### 1. `cpt_collection` — Default Free Baseline

- Native WordPress Custom Post Type per repeatable collection.
- Taxonomy ahol kategorizáció szükséges (pl. gallery image category, product category).
- ACF Free csak egyszerű mezőkre a CPT item-eken (text, image, textarea).
- Response builder: `WP_Query` / `get_posts()` → normalizált SiteData tömb.
- Alkalmas: galéria képek, márkák, csapattagok, termékek, szolgáltatások, állatok, és egyéb user-managed, unbounded, repeatable tartalom.

#### 2. `fixed_slots` — Bounded Landing Page Content

- Kizárólag kis, bounded tartalom (max 3-7 item).
- ACF Free numbered fields repeater nélkül.
- Nem használható large, unbounded, vagy kliens által szabadon bővíthető kolekciókra.
- Példák: max 4 statisztika, ~5-7 nyitvatartási sor, ~3-5 bekezdés, ~5-10 service label.

#### 3. `acf_repeater_optional` — Optional Future Pro Source

- Opcionális jövőbeli enhancement.
- ACF Pro szükséges.
- Nem része a free baseline-nak.
- Ugyanazt a SiteData output shape-et kell produkálnia, mint a `cpt_collection` vagy `fixed_slots`.
- Dokumentálható mint opcionális source adapter strategy.

### Classification Rule

Minden repeater field-et content karakterisztika alapján kell besorolni:

| Jellemző | Strategy |
|----------|----------|
| User-managed, unbounded, kategorized, reusable list | `cpt_collection` |
| Bounded, kis, egyetlen szekción belüli tartalom | `fixed_slots` |
| Premium simplification, opcionális | `acf_repeater_optional` |

### SiteData Stability Rule

A frontend SiteData shape változatlan marad. A source strategy csere a WordPress builder/mapper mögött történik. ACF repeater → CPT collection váltás nem igényel frontend section prop módosítást.

### Builder Abstraction Boundary

Az sp-infra `spektra_get_field()` wrapper az ACF `get_field()` API-t hívja. Ez field-szintű ACF source helper — a repeater array-t közvetlenül adja vissza.

CPT collection strategy-nál a repeatable content nem `spektra_get_field()`-en keresztül jön, hanem `WP_Query` / `get_posts()` alapú collection loading-gel. A builder-ek felelőssége eldönteni, melyik source-t használják:

- Scalar/image mezők → `spektra_get_field()` (ACF Free OK)
- Repeatable collection → `WP_Query` CPT loading (ACF Free OK, no repeater dependency)

### Seed Tooling Impact

A jelenlegi seed pipeline (`import-seed.php`, `dump-acf.php`) explicit repeater-aware logikát tartalmaz:
- `import-seed.php`: `'repeater'` kind detection, recursive row walker, `update_field()` ACF API
- `dump-acf.php`: repeater-aware recursive dump

CPT collection seed flow: `wp_insert_post()` + `update_post_meta()` + ACF Free simple fields. Ez új seed path — a meglévő repeater seed logic a transition periódusban megmarad.

## Why It Exists

A Spektra platform ökoszisztéma free/open-source kompatibilis baseline-ként pozicionált. Az ACF Pro mint hard dependency sérti ezt. A source strategy abstrakció lehetővé teszi, hogy a free baseline ACF Pro nélkül működjön, miközben a Pro opcionálisan elérhető marad.

## Boundaries and Guardrails

- WordPress/CPT/ACF logika NEM kerülhet az sp-platform core-ba (DR-001).
- SiteData a kanonikus frontend contract (DR-002).
- ACF Pro nem baseline requirement.
- BenettCar/Exotica specifikus logika nem kerülhet shared infra core-ba (DR-005).
- A kliens overlay felelős a builder implementációért — a strategy választás is overlay szintű.

## Related Concepts

- DR-001: WordPress = opcionális content source
- DR-002: SiteData mint kanonikus contract
- DR-003: Fixed Field Groups (ACF Free) — korrigálva, lásd DR-015
- DR-012: Kétszintű ACF modell
- DR-013: Seed pipeline
- `concepts/seed-pipeline.md` — Seed pipeline tartalom szinkronizáció
- `concepts/wordpress-mapper-boundary.md` — WP REST → SiteData mapper boundary

## Open Questions

1. **CPT naming convention** — Milyen CPT slug konvenció? `spk_gallery_item`, `bc_brand`, prefix nélküli? Implementation decision.
2. **Taxonomy strategy** — Shared taxonomy vs per-CPT taxonomy? Implementation decision.
3. **Seed pipeline CPT support** — `wp_insert_post()` alapú seed flow szükséges. Későbbi fázis.
4. **ACF Pro transition deadline** — Explicit deadline szükséges, vagy elég a per-client migration plan?
5. **fixed_slots implementation** — ACF Free numbered fields (`opening_hours_1`, `opening_hours_2`) vagy textarea + split?
6. **sp-infra helper scope** — `spektra_get_field()` maradjon ACF-only, CPT loading külön helper? Vagy közös facade?
