# Phase 12.2 — Repeatable Content Baseline Correction

## Purpose

A Phase 12.2 a repeatable content source strategy governance correction — az ACF Pro repeater baseline-függőség dokumentálása és a free baseline helyreállítási terv meghatározása. A Phase 12.1 (Second Client Validation) során validált multi-client implementáció feltárta, hogy mindkét kliens inception óta ACF Pro repeater mezőket használ, ami ellentmond DR-003 ("ACF Free elég") governance-nek.

---

## Kontextus

- DR-003 implicit módon Free sufficiency-t állított
- Mindkét kliens (sp-benettcar, sp-exotica) inception óta ACF Pro repeater-t használ
- Ez nem drift — a governance és az implementáció sosem volt konzisztens
- 13 repeater field összesen (8 benettcar, 5 exotica), 11 szekción keresztül

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

- 6 section builder, ebből 5 használ repeater adatot
- 5 repeater field összesen
- Érintett szekciók: eb-gallery, eb-products, eb-animals, eb-contact, eb-about

---

## Repeater Classification Table

| Client | Section | Repeater field | Strategy | Rationale |
|--------|---------|---------------|----------|-----------|
| benettcar | bc-gallery | `bc_gallery_images` | `cpt_collection` | Unbounded, categorized image collection |
| benettcar | bc-brand | `bc_brand_brands` | `cpt_collection` | User-managed brand list, logo + metadata |
| benettcar | bc-team | `bc_team_members` | `cpt_collection` | User-managed team, image + contact |
| benettcar | bc-services | `bc_services_services` | ~~`cpt_collection`~~ → `fixed_slots` (P13.1) | Service list, icon + description. CPT pilot valid, superseded for bounded landing UX by slot-based admin. |
| benettcar | bc-service | `bc_service_services` | `fixed_slots` ✅ P13.2 | Bounded service labels → textarea+split |
| benettcar | bc-service | `bc_service_brands` | `fixed_slots` ✅ P13.2 | Bounded brand names → textarea+split |
| benettcar | bc-about | `bc_about_content` | `fixed_slots` ✅ P13.2 | ~3-5 paragraphs → textarea+split |
| benettcar | bc-about | `bc_about_stats` | `fixed_slots` ✅ P13.2 | ~4 items → slot fields (value+label) |
| exotica | eb-gallery | `eb_gallery_images` | `cpt_collection` | Unbounded, categorized image collection |
| exotica | eb-products | `eb_products_products` | `cpt_collection` | User-managed product list, image + metadata |
| exotica | eb-animals | `eb_animals_animals` | `cpt_collection` | User-managed animal list, image + metadata |
| exotica | eb-contact | `eb_contact_opening_hours` | `fixed_slots` | ~5-7 text lines, single sub-field |
| exotica | eb-about | `eb_about_values` | `fixed_slots` | ~3-7 bounded value strings, single text sub-field |

### Summary

- **`cpt_collection`**: 7 field (54%) — CPT szükséges; taxonomy csak ahol kategorizáció indokolt
- **`fixed_slots`**: 6 field (46%) — ACF Free-kompatibilis bounded mezők, nincs repeater dependency

---

## Operational Priority

**Technikai pilot**: `eb-contact.opening_hours` vagy más kis `fixed_slots` eset — lowest-risk, validációs célú.

**Operatív prioritás**: BenettCar handover stabilitás. Különösen a gallery / brand / team editorial experience kritikus — ezek a `cpt_collection` jelöltek, amelyek a legtöbb szerkesztői interakciót kapják. A technikai pilot nem helyettesíti az operatív prioritást — a BenettCar kliensnél a repeater → CPT migration sorrendje az editorial impact alapján dől el, nem a technikai egyszerűség alapján.

---

## Implementation Phases (proposal)

1. **P12.2** (current): Governance correction dokumentáció, classification, DR-015
2. **P12.3** (future): First Runtime Migration Planning / Pilot
   - Candidate A: `eb-contact.opening_hours` vagy `eb-about.values` → `fixed_slots` (lowest-risk technikai pilot)
   - Candidate B: BenettCar gallery / brand / team migration planning (operatív prioritás handover-hez)
   - Végleges target explicit döntést igényel implementáció előtt
3. **P12.4+** (future): CPT collection migrations a P12.3 döntés alapján

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

## Closeout

P12.2 lezárva mint dokumentációs/governance fázis. Runtime implementáció a P12.3+ fázisokra halasztva.

Következő: `phases/phase-12.3.md`

---

## Open Questions

1. **CPT naming convention** — Milyen CPT slug konvenció? `spk_gallery_item`, `bc_brand`, prefix nélküli? Implementation decision a P12.3+ fázisban.
2. **BenettCar migration sorrend** — Gallery / brand / team editorial impact alapján, vagy technikai egyszerűség alapján?
3. **fixed_slots implementation** — ACF Free numbered fields (`opening_hours_1`, `opening_hours_2`) vagy textarea + split?
4. **ACF Pro transition deadline** — Explicit deadline szükséges, vagy elég a per-client migration plan?
5. **Seed pipeline CPT support** — `wp_insert_post()` alapú seed flow ideje a P12.3+ fázisra tervezett.
