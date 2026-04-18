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
- P12.3: Option C split tracks aktív — P12.3a fixed_slots pilot planning, P12.3b BenettCar cpt_collection design
- Runtime migration NEM indult el
- ACF Pro NEM baseline requirement
- SiteData shape stabil marad

---

## Phase Map

| Phase | Cím | Státusz | Típus |
|-------|------|---------|-------|
| P12.1 | Second Client Validation + Platform Elevation Discovery | CLOSED | Runtime + docs |
| P12.2 | Repeatable Content Baseline Correction | CLOSED | Docs / governance |
| P12.3 | First Runtime Migration Planning / Pilot | ACTIVE | Planning only |
| P12.3a | fixed_slots Technical Pilot Planning | READY | Planning only |
| P12.3b | BenettCar cpt_collection Design | NOT STARTED | Design only |
| P12.4 | First Runtime Migration Implementation | BLOCKED | Runtime |
| P12.5 | Seed Pipeline Source Strategy Support | PLANNED | Design + runtime |
| P12.6 | BenettCar Handover Hardening | PLANNED | Design + runtime |
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

#### P12.3b — BenettCar cpt_collection Design

**TODO:**
- [ ] Első target ajánlás:
  - `bc-gallery.images`
  - `bc-brand.brands`
  - `bc-team.members`
- [ ] CPT naming convention javaslat
- [ ] Collection loading pattern javaslat
- [ ] Builder output shape
- [ ] Seed strategy impact
- [ ] SiteData parity validation plan
- [ ] Migration/rollback plan
- [ ] Handover impact assessment
- [ ] Implementációs prompt készültségi értékelés

**Lezárási feltétel:** Design ajánlás dokumentálva. A végleges cpt_collection target kiválasztás explicit platform-owner jóváhagyást igényel.

---

### P12.4 — First Runtime Migration Implementation [BLOCKED]

**Státusz:** blokkolt — P12.3 target döntés szükséges

**Cél:**
- az első elfogadott source strategy migration végrehajtása
- SiteData shape stabilitás bizonyítása runtime cserénél

**Érintett repók:** sp-exotica; sp-infra csak akkor, ha a shared seed tooling módosítása szükséges.

**TODO:**
- [ ] Végrehajtás kizárólag elfogadott implementációs prompt alapján
- [ ] ACF field group módosítás (repeater → bounded fields)
- [ ] Builder módosítás (repeater array read → bounded field read)
- [ ] Seed pipeline update (ha szükséges)
- [ ] SiteData parity validation (output nem változhat)
- [ ] Rollback verification
- [ ] Before/after dokumentáció
- [ ] CHANGELOG és phase note frissítés

**Guardrail:** SiteData shape változatlan. Ha a parity validation fail, a csere nem mehet élesbe.

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

### P12.6 — BenettCar Handover Hardening [PLANNED]

**Státusz:** tervezett

**Cél:**
- kliens-facing WordPress admin experience stabilizálása
- editorial UX javítása a magas interakciójú tartalomhoz

**TODO:**
- [ ] Gallery / brand / team editorial UX prioritásozása
- [ ] Első cpt_collection migration target döntés (P12.3b ajánlás alapján)
- [ ] Első BenettCar CPT runtime migration végrehajtása
- [ ] Admin guide írása
- [ ] Endpoint parity validáció
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
| `wp-integration-plan-v4.md` DR-015 | Decision Record | Proposed |
