# Phase 12.2 — Repeatable Content Baseline Correction

## Purpose

A Phase 12.2 a repeatable content source strategy governance correction — az ACF Pro repeater baseline-függőség dokumentálása és a free baseline helyreállítási terv meghatározása. A Phase 12.1 (Second Client Validation) során validált multi-client implementáció feltárta, hogy mindkét kliens inception óta ACF Pro repeater mezőket használ, ami ellentmond DR-003 ("ACF Free elég") governance-nek.

---

## Kontextus

- DR-003 implicit módon Free sufficiency-t állított
- Mindkét kliens (sp-benettcar, sp-exotica) inception óta ACF Pro repeater-t használ
- Ez nem drift — a governance és az implementáció sosem volt konzisztens
- 12 repeater field összesen (8 benettcar, 4 exotica), 10 szekción keresztül

## Canonical Concept

→ `concepts/repeatable-content-source-strategy.md`

---

## Transition Rules

1. **Új repeater field**: TILOS a free baseline-ban. Csak `cpt_collection` vagy `fixed_slots`.
2. **Meglévő repeater field**: Működik, de classification szükséges → migration plan.
3. **SiteData shape**: Source strategy csere NEM módosíthatja a frontend SiteData contract-ot.
4. **Client overlay felelősség**: A strategy választás és a builder implementáció kliens overlay szintű döntés.
5. **Seed pipeline**: A meglévő repeater-aware seed logic a transition periódusban megmarad. CPT seed flow későbbi fázis.

---

## Client Impact

### sp-benettcar

- 10 section builder, ebből 6 használ repeater adatot
- 8 repeater field összesen
- Érintett szekciók: bc-gallery, bc-brand, bc-team, bc-services, bc-service, bc-about

### sp-exotica

- 6 section builder, ebből 4 használ repeater adatot
- 4 repeater field összesen
- Érintett szekciók: eb-gallery, eb-products, eb-animals, eb-contact

---

## Repeater Classification Table

| Client | Section | Repeater field | Strategy | Rationale |
|--------|---------|---------------|----------|-----------|
| benettcar | bc-gallery | `bc_gallery_images` | `cpt_collection` | Unbounded, categorized image collection |
| benettcar | bc-brand | `bc_brand_brands` | `cpt_collection` | User-managed brand list, logo + metadata |
| benettcar | bc-team | `bc_team_members` | `cpt_collection` | User-managed team, image + contact |
| benettcar | bc-services | `bc_services_services` | `cpt_collection` | Service list, icon + description |
| benettcar | bc-service | `bc_service_services` | `fixed_slots` | Bounded service labels, single sub-field |
| benettcar | bc-service | `bc_service_brands` | `fixed_slots` | Bounded brand names, string array |
| benettcar | bc-about | `bc_about_content` | `fixed_slots` | ~3-5 paragraphs, single sub-field |
| benettcar | bc-about | `bc_about_stats` | `fixed_slots` | ~4 items, value + label pair |
| exotica | eb-gallery | `eb_gallery_images` | `cpt_collection` | Unbounded, categorized image collection |
| exotica | eb-products | `eb_products_products` | `cpt_collection` | User-managed product list, image + metadata |
| exotica | eb-animals | `eb_animals_animals` | `cpt_collection` | User-managed animal list, image + metadata |
| exotica | eb-contact | `eb_contact_opening_hours` | `fixed_slots` | ~5-7 text lines, single sub-field |

### Summary

- **`cpt_collection`**: 7 field (58%) — új CPT + taxonomy szükséges
- **`fixed_slots`**: 5 field (42%) — ACF Free numbered fields, nincs repeater dependency

---

## First Migration Target

`eb-contact.opening_hours` — a legegyszerűbb fixed_slots jelölt:
- Egyetlen sub-field (szöveg sor)
- ~5-7 bounded item
- Nincs image, nincs nested data
- Exotica még fejlesztés közben → kisebb kockázat

---

## Operational Priority

**Technikai pilot**: `eb-contact.opening_hours` vagy más kis `fixed_slots` eset — lowest-risk, valíDációs célú.

**Operatív prioritás**: BenettCar handover stabilitás. Különösen a gallery / brand / team editorial experience kritikus — ezek a `cpt_collection` jelöltek, amelyek a legtöbb szerkesztői interakciót kapják. A technikai pilot nem helyettesíti az operatív prioritást — a BenettCar kliensnél a repeater → CPT migration sorrendje az editorial impact alapján dől el, nem a technikai egyszerűség alapján.

---

## Implementation Phases (proposal)

1. **P12.2** (current): Governance correction dokumentáció, classification, DR-015
2. **P12.3** (future): First migration — `eb_contact_opening_hours` → `fixed_slots`
3. **P12.4+** (future): CPT collection migrations — gallery, brands, team, stb.

---

## Deliverables

- [x] `concepts/repeatable-content-source-strategy.md` — canonical concept
- [x] `phases/phase-12.2.md` — this file
- [x] DR-015 in `wp-integration-plan-v4.md`
- [x] DR-003 status update
- [x] CHANGELOG update

---

## Done State

P12.2 lezárási feltétel:
1. Canonical concept doc (`repeatable-content-source-strategy.md`) érvényes és review-olt
2. Minden meglévő repeater field classifikálva (classification table teljes)
3. DR-015 Proposed státuszban a wp-integration-plan-v4.md-ben
4. DR-003 cross-reference hozzáadva
5. Transition rules dokumentálva és elfogadva
6. Runtime kód NEM módosult — ez kizárólag dokumentációs korrekció

---

## Open Questions

1. **CPT naming convention** — Milyen CPT slug konvenció? `spk_gallery_item`, `bc_brand`, prefix nélküli? Implementation decision a P12.3+ fázisban.
2. **BenettCar migration sorrend** — Gallery / brand / team editorial impact alapján, vagy technikai egyszerűség alapján?
3. **fixed_slots implementation** — ACF Free numbered fields (`opening_hours_1`, `opening_hours_2`) vagy textarea + split?
4. **ACF Pro transition deadline** — Explicit deadline szükséges, vagy elég a per-client migration plan?
5. **Seed pipeline CPT support** — `wp_insert_post()` alapú seed flow ideje a P12.3+ fázisra tervezett.
