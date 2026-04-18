---
title: Phase 12 — Platform Hardening Roadmap
status: draft
type: phase-roadmap
canonical: true
last_updated: 2026-04-18
---

# Phase 12 — Platform Hardening Roadmap

## Purpose

A Phase 12 hátralévő munkájának összefoglalása a multi-client validáció (P12.1), a repeatable content governance korrekció (P12.2) és az Option C split-track döntés (P12.3) után.

## Current State

- P12.1: Platform Elevation / No Copy-Paste Client irány dokumentálva
- P12.2: Repeatable Content Baseline Correction lezárva
- P12.3: Option C split tracks — P12.3a CLOSED (fixed_slots pilot), P12.3b CLOSED (cpt_collection design + runtime)
- P12.4: fixed_slots runtime migration lezárva (eb-about.values, parity PASS)
- P12.6: első cpt_collection runtime migration végrehajtva (bc-services.services, parity PASS, transition hardening alkalmazva)
- P13.1: bc-services admin refactored to Slot-Based ACF Free Admin (CPT collection superseded for bounded landing content UX)
- Corrected source strategy rule: bounded landing content → slot-based fixed collection, unbounded managed content → cpt_collection
- ACF Pro NEM baseline requirement
- SiteData shape stabil marad

---

## Phase Map

| Phase | Cím | Státusz | Típus |
|-------|------|---------|-------|
| P12.1 | Second Client Validation + Platform Elevation Discovery | CLOSED | Runtime + docs |
| P12.2 | Repeatable Content Baseline Correction | CLOSED | Docs / governance |
| P12.3 | First Runtime Migration Planning / Pilot | ACTIVE | Planning only |
| P12.3a | fixed_slots Technical Pilot Planning | CLOSED | Planning only |
| P12.3b | BenettCar cpt_collection Design | CLOSED | Design + runtime |
| P12.4 | First Runtime Migration Implementation | CLOSED | Runtime |
| P12.5a | Seed Import Failure Diagnosis | CLOSED | Runtime fix |
| P12.5 | Seed Pipeline Source Strategy Support | PLANNED | Design + runtime |
| P12.6 | BenettCar Handover Hardening | CLOSED | Design + runtime |
| P13.1 | Slot-Based Admin Refactor — bc-services | CLOSED | Runtime + admin UX |
| P12.7 | Platform Scaffold / Generator Prep | DEFERRED | Design |

---

## Phase 12 Tracks

### P12.1 — Second Client Validation + Platform Elevation Discovery [CLOSED]

**Státusz:** dokumentált

**Cél:**
- sp-exotica létrehozása, multi-client proof-of-execution
- copy-paste client creation problémájának feltárása
- shared vs client-specific fájlok osztályozása (A/B/C/D)
- scaffold/generator irány előkészítése

**Eredmény:**
- [x] sp-exotica seed pipeline: 39/39 parity, 20/20 endpoint PASS
- [x] Platform Elevation concept (A/B/C/D file classification)
- [x] No Copy-Paste Client guardrail
- [x] Seed Pipeline Blocker Taxonomy (B1-B5)

**Tanulság:** Copy-paste nem skálázódik. 6 fájl 100%-an azonos a kliensek közt. ACF Pro repeater dependency felszínre került.

**TODO:**
- [ ] Jelenlegi docs státusz megerősítése
- [ ] Stratégiai inputként használni a későbbi scaffold/generator munkához (P12.7)

---

### P12.2 — Repeatable Content Baseline Correction [CLOSED]

**Státusz:** lezárva

**Cél:**
- free WordPress baseline NEM függhet ACF Pro Repeater-től
- source strategy-k definiálása:
  - `cpt_collection` — default free baseline (CPT; taxonomy csak ahol kategorizáció indokolt)
  - `fixed_slots` — bounded kis tartalom (ACF Free-kompatibilis)
  - `acf_repeater_optional` — opcionális jövőbeli Pro source

**Eredmény:**
- [x] Repeatable Content Source Strategy concept → `status: accepted`
- [x] 13 repeater field / 11 szekció classifikálva
- [x] DR-003 korrekció, DR-015 hozzáadva
- [x] Transition rules dokumentálva
- [x] Runtime kód NEM módosult

**Classification összefoglaló:**

| Strategy | Darab | Arány | Jelleg |
|----------|-------|-------|--------|
| `cpt_collection` | 7 | 54% | CPT; taxonomy csak ahol kategorizáció indokolt |
| `fixed_slots` | 6 | 46% | ACF Free-kompatibilis bounded mezők |

**Repeater field térkép:**

| # | Client | Section | Field | Strategy |
|---|--------|---------|-------|----------|
| 1 | benettcar | bc-gallery | `bc_gallery_images` | cpt_collection |
| 2 | benettcar | bc-brand | `bc_brand_brands` | cpt_collection |
| 3 | benettcar | bc-team | `bc_team_members` | cpt_collection |
| 4 | benettcar | bc-services | `bc_services_services` | cpt_collection |
| 5 | benettcar | bc-service | `bc_service_services` | fixed_slots |
| 6 | benettcar | bc-service | `bc_service_brands` | fixed_slots |
| 7 | benettcar | bc-about | `bc_about_content` | fixed_slots |
| 8 | benettcar | bc-about | `bc_about_stats` | fixed_slots |
| 9 | exotica | eb-gallery | `eb_gallery_images` | cpt_collection |
| 10 | exotica | eb-products | `eb_products_products` | cpt_collection |
| 11 | exotica | eb-animals | `eb_animals_animals` | cpt_collection |
| 12 | exotica | eb-contact | `eb_contact_opening_hours` | fixed_slots |
| 13 | exotica | eb-about | `eb_about_values` | fixed_slots |

**TODO:**
- Nincs runtime TODO
- Governance baseline a P12.3+ számára

---

### P12.3 — First Runtime Migration Planning / Pilot [ACTIVE]

**Státusz:** aktív

**Döntés:** Option C — Split Tracks.

#### P12.3a — fixed_slots Technical Pilot Planning

**Selected target:** `eb-about.values` (sp-exotica)

**TODO:**
- [x] Target kiválasztás: `eb-about.values`
- [x] Field representation döntés: textarea + split
- [x] Builder mapping plan
- [x] Seed impact
- [x] SiteData parity validation plan
- [x] Rollback plan
- [x] Implementációs prompt produkálása: **READY**

**Lezárási feltétel:** Implementációs prompt biztonságosan megírható. Runtime implementáció külön explicit promptot igényel.

#### P12.3b — BenettCar cpt_collection Design [DESIGN COMPLETE]

**Selected target:** `bc-services.services` (sp-benettcar)

**Scope kiegészítés:** `bc-services.services` nem volt az eredeti Candidate B listában, de P12.2 classifikáció alapján `cpt_collection` stratégiájú. Legalacsonyabb komplexitás: 3 scalar mező, nincs image, nincs taxonomy.

**Design döntések:**
- CPT slug: `sp_bc_service` (14 char)
- Loading: kliens-lokális `spektra_bc_get_services()`
- SiteData shape: változatlan
- Seed: `kind: cpt_collection` → P12.5 scope

**Design dokumentum:** `implementation/p12-3b-benettcar-cpt-collection-design.md`

**Deferred targets:** `bc-brand.brands`, `bc-team.members`, `bc-gallery.images` (image/taxonomy komplexitás)

**Lezárási feltétel:** Design ajánlás dokumentálva. Runtime implementáció külön explicit promptot igényel.

---

### P12.4 — First Runtime Migration Implementation [CLOSED]

**Státusz:** lezárva — commit `b3f0e56` (sp-exotica), parity validation PASS

**Cél:**
- az első elfogadott source strategy migration végrehajtása
- SiteData shape stabilitás bizonyítása runtime cserénél

**Érintett repók:** sp-exotica (sp-infra NEM módosult)

**TODO:**
- [x] Végrehajtás elfogadott implementációs prompt alapján
- [x] ACF field group módosítás (repeater → textarea)
- [x] Builder módosítás (`spektra_split_textarea()` lokális helper)
- [x] Seed mapping update (repeater → scalar)
- [x] SiteData parity validation: **PASS** — 3 value string, pontos egyezés
- [x] Before/after dokumentáció
- [x] CHANGELOG és phase note frissítés

**Eredmény:** Az első fixed_slots migration pattern bizonyítva. ACF Pro Repeater kiváltható ACF Free textarea + builder split-tel a SiteData contract megtartásával.

---

### P12.5a — Seed Import Failure Diagnosis [CLOSED]

**Státusz:** lezárva

**Cél:**
- B5 blocker (seed import 35 FAIL / 0 success) diagnosztizálása és javítása
- A seed pipeline megbízhatósága a további migrációk előfeltétele

**Root cause:** `update_field()` → WordPress `update_metadata()` returns `false` when value is unchanged (by design). Az import script minden `false`-t hibának tekintett.

**Fix:** `import-seed.php` — when `update_field` returns `false`, verify via `get_field()` + `field_values_match()` helper. sp-infra módosítás.

**TODO:**
- [x] `import-seed.php` teljes kódbázis áttekintés
- [x] Root cause azonosítás (WP `update_metadata` no-op = `false`)
- [x] `field_values_match()` helper implementálása
- [x] Validálás: 37/37 field OK, 0 FAIL
- [x] B5 troubleshooting entry frissítése → RESOLVED

**Eredmény:** Seed import pipeline megbízható. Ismételt import nem dob false positive FAIL-eket.

---

### P12.5 — Seed Pipeline Source Strategy Support [PLANNED]

**Státusz:** tervezett

**Cél:**
- seed tooling kiterjesztése a repeater-aware ACF flow-n túl
- fixed_slots és cpt_collection seed path-ek támogatása

**TODO:**
- [ ] Jelenlegi seed feltételezések dokumentálása
- [ ] Source-strategy-aware seed modell design
- [ ] `update_field()` támogatás ACF simple field-ekhez (fixed_slots)
- [ ] `wp_insert_post()` / `update_post_meta()` támogatás CPT collection-ökhöz
- [ ] dump/export viselkedés definiálása az új source-okra
- [ ] Meglévő repeater seed logic backward compatibility megőrzése

---

### P12.6 — BenettCar Handover Hardening [IN PROGRESS]

**Státusz:** első cpt_collection runtime migration végrehajtva — `bc-services.services`

**Cél:**
- kliens-facing WordPress admin experience stabilizálása
- editorial UX javítása a magas interakciójú tartalomhoz

**Completed:**
- [x] Első cpt_collection migration target döntés: `bc-services.services` (P12.3b ajánlás alapján)
- [x] CPT `sp_bc_service` regisztráció + ACF field group
- [x] `spektra_bc_get_services()` kliens-lokális loader
- [x] Builder CPT-first + repeater fallback
- [x] Migration script: 3/3 repeater row → CPT post
- [x] Endpoint parity validáció: **PASS**

**TODO:**
- [ ] Gallery / brand / team editorial UX prioritásozása
- [ ] Admin guide írása
- [ ] Handover-safe WP admin flow megerősítése
- [ ] Maradék CPT collection migration-ök (további benettcar + exotica field-ek)

**Érintett cpt_collection field-ek (7 db):**

| # | Client | Field | Complexitás |
|---|--------|-------|-------------|
| 1 | benettcar | `bc_gallery_images` | Magas — image handling, taxonomy |
| 2 | benettcar | `bc_brand_brands` | Közepes — logo + metadata |
| 3 | benettcar | `bc_team_members` | Közepes — image + contact fields |
| 4 | benettcar | `bc_services_services` | Közepes — icon + description |
| 5 | exotica | `eb_gallery_images` | Magas — image handling, taxonomy |
| 6 | exotica | `eb_products_products` | Közepes — image + metadata |
| 7 | exotica | `eb_animals_animals` | Közepes — image + metadata |

**Sorrend:** BenettCar editorial impact alapján prioritásozott (handover stabilitás), végleges sorrend a P12.3b ajánlásból.

---

### P12.7 — Platform Scaffold / Generator Prep [DEFERRED]

**Státusz:** halasztott

**Cél:**
- jövőbeli kliens scaffold/generator előkészítése kizárólag a source strategy pattern-ök bizonyítása után

**TODO:**
- [ ] NEM generálni klienseket instabil pattern-ekből
- [ ] Scaffold requirements levezetése a P12.3–P12.6 eredményekből
- [ ] Schema/generator irány előkészítése később
- [ ] P12.1 Platform Elevation / No Copy-Paste Client input felhasználása

---

## Nyitott Kérdések (cross-phase)

| # | Kérdés | Fázis | Státusz |
|---|--------|-------|---------|
| 1 | CPT naming convention (`spk_gallery_item`, `bc_brand`, prefix nélküli?) | P12.3b | Döntés szükséges |
| 2 | BenettCar migration sorrend (editorial impact vs technikai egyszerűség) | P12.3b / P12.6 | Döntés szükséges |
| 3 | ~~fixed_slots field representation~~ | P12.3a | RESOLVED: textarea + split |
| 4 | ACF Pro transition deadline (explicit vs per-client migration plan) | P12.6 | Döntés szükséges |
| 5 | Seed pipeline CPT support (`wp_insert_post()` flow) | P12.5 | Design szükséges |
| 6 | sp-infra helper scope (`spektra_get_field()` vs közös facade) | P12.5 | Design szükséges |
| 7 | Taxonomy strategy (shared vs per-CPT) | P12.6 | Design szükséges |

---

## Guardrails (cross-phase)

1. WordPress/CPT/ACF logika NEM kerülhet az sp-platform core-ba (DR-001)
2. SiteData a kanonikus frontend contract — változatlan marad minden migration során (DR-002)
3. Frontend section props változatlanok
4. ACF Pro opcionális, NEM free baseline requirement
5. Meglévő kliensek megtarthatják a repeater-eket a migration-ig
6. Runtime implementáció explicit promptot igényel
7. Kliens-specifikus logika kliens overlay-ben marad
8. sp-infra reusable marad, nem client-hardcoded
9. Scaffold/generator NEM indulhat source strategy pattern-ök bizonyítása előtt

---

## Done State — Phase 12

Phase 12 lezárható, ha:
1. Legalább egy fixed_slots migration pattern bizonyított vagy explicit módon elvetett
2. Legalább egy cpt_collection design elfogadott
3. Seed strategy impact dokumentált
4. BenettCar handover blokkolók feloldottak vagy explicit módon halasztottak
5. Platform scaffold/generator kockázatok dokumentáltak
6. Nincs dokumentálatlan free baseline ACF Pro dependency

---

## Dokumentum és Artifact Index

| Dokumentum | Típus | Státusz |
|------------|-------|---------|
| `concepts/repeatable-content-source-strategy.md` | Concept | accepted |
| `concepts/platform-elevation.md` | Concept | draft |
| `guardrails/no-copy-paste-client.md` | Guardrail | draft |
| `troubleshooting/seed-pipeline-blockers.md` | Troubleshooting | draft |
| `phases/phase-12.1.md` | Phase index | closed |
| `phases/phase-12.2.md` | Phase index | closed |
| `phases/phase-12.3.md` | Phase index | active |
| `wp-integration-plan-v4.md` DR-003 | Decision Record | Korrigálva |
| `implementation/p12-4-eb-about-values-migration.md` | Implementation prompt | ready |
| `wp-integration-plan-v4.md` DR-015 | Decision Record | Proposed |
