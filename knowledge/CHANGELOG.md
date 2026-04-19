# Knowledge Changelog

Minden knowledge dokumentum változás naplója. Keep a Changelog formátum.

## [Unreleased]

### Added
- P13.4: BenettCar seed mapping aligned with slot-based ACF Free model — 8 stale `kind: 'repeater'` mappings replaced with slot/textarea extractors (makeSlotScalars, makeSlotImages, joinLines). P13.3 migration unused slot clearing added. 194-field seed export validated.
- P13.3: BenettCar image-bearing repeaters migrated to slot-based ACF Free fields — bc_gallery_images (max 10 image slots), bc_brand_brands (max 10 brand slots), bc_team_members (max 8 member slots). 3/3 parity PASS.
- P13.2: BenettCar fixed_slots batch migration — bc_about_content (textarea+split), bc_about_stats (slot fields), bc_service_services (textarea+split), bc_service_brands (textarea+split). 4/4 parity PASS.
- `implementation/p13-1-bc-services-slot-based-admin-refactor.md` — P13.1: bc-services cpt_collection pilot superseded by Slot-Based ACF Free Admin for BenettCar handover UX. 6 slot fields, CPT hidden, repeater retained, SiteData shape unchanged.
- `implementation/p12-3b-benettcar-cpt-collection-design.md` — P12.3b design: bc-services.services cpt_collection target. CPT slug `sp_bc_service`, kliens-lokális loader, SiteData shape változatlan, seed deferred P12.5-re. Design only — runtime implementáció NEM tartozik ide.
- `implementation/p12-4-eb-about-values-migration.md` — P12.4 implementációs prompt: eb-about.values ACF Repeater → textarea+split migration. Teljes before/after kód, data flow, seed flow, parity validation, rollback plan. **CLOSED:** commit `b3f0e56` (sp-exotica), parity validation PASS.

### Changed
- `phases/phase-12-strategy-overview.md` — P13.4: Current State + Phase Map frissítve, seed pipeline aligned with slot model
- `phases/phase-12-strategy-overview.md` — P13.3: Current State + Phase Map frissítve, BenettCar fixed_slots 8/8 migrated
- `phases/phase-12.2.md` — bc-gallery + bc-brand + bc-team classification annotated: cpt_collection → fixed_slots ✅ P13.3
- `phases/phase-12-strategy-overview.md` — P13.2: Current State + Phase Map frissítve, BenettCar fixed_slots 5/5 migrated
- `phases/phase-12.2.md` — bc-about + bc-service classification annotated: ✅ P13.2
- `phases/phase-12-strategy-overview.md` — P13.1 slot-based admin refactor: Current State + Phase Map frissítve, P12.6 CLOSED, P13.1 CLOSED, corrected source strategy rule
- `phases/phase-12.2.md` — bc-services.services classification annotated: cpt_collection → fixed_slots (P13.1)
- `phases/phase-12-strategy-overview.md` — Current State frissítve: runtime migration státusz pontosítva (P12.4 fixed_slots CLOSED, P12.6 cpt_collection IN PROGRESS, transition hardening alkalmazva). "Runtime migration NEM indult el" téves állítás törölve.
- `phases/phase-12-strategy-overview.md` — P12.3b CLOSED (runtime migration executed), P12.6 IN PROGRESS, first cpt_collection runtime: bc-services.services, parity PASS
- `phases/phase-12.3.md` — P12.3b cpt_collection design target selected: `bc-services.services`. Scope kiegészítés dokumentálva (P12.2 classifikáció alapján). Design TODO-k lezárva, implementációs prompt pending.
- `phases/phase-12-strategy-overview.md` — P12.3a CLOSED, P12.4 CLOSED (parity PASS), Phase Map frissítve
- `phases/phase-12.3.md` — P12.3a fixed_slots pilot planning: target `eb-about.values` kiválasztva, textarea+split field representation elfogadva, builder mapping plan, seed impact, SiteData parity validation plan, rollback plan, implementációs prompt készültségi értékelés: READY. Pre-existing mapper bug dokumentálva. Helper placement döntés: lokális sp-exotica.
- `phases/phase-12-strategy-overview.md` — P12.3a TODO-k lezárva, Phase Map P12.3a READY / P12.4 READY, Open Question #3 resolved, Artifact Index bővítve
- `concepts/repeatable-content-source-strategy.md` — P12.2 Repeatable Content Source Strategy: 3 source strategy (cpt_collection / fixed_slots / acf_repeater_optional), SiteData stability rule, builder abstraction boundary, 13 repeater classification
- `phases/phase-12.2.md` — Phase 12.2 index: Repeatable Content Baseline Correction (governance mismatch DR-003, transition rules, classification table, migration plan)
- `phases/phase-12.3.md` — Phase 12.3 index: First Runtime Migration Planning / Pilot (Candidate A/B/C, decision criteria, required output, guardrails)
- `concepts/platform-elevation.md` — P12.1 Platform Elevation: A/B/C/D fájl-osztályozási modell, elevation path (P1/P2/P3), sp-exotica vs sp-benettcar összehasonlításból
- `guardrails/no-copy-paste-client.md` — P12.1 No Copy-Paste Client guardrail: kliens létrehozás nem fájlmásolással, döntési fa (A/B/C/D)
- `troubleshooting/seed-pipeline-blockers.md` — P12.1 Seed Pipeline Blocker Taxonomy: B1–B5 (tsx, Unsplash 404, @types/node, ACF false, nem validált URL)
- `phases/phase-12.1.md` — Phase 12.1 index: Second Client Validation (sp-exotica seed pipeline, multi-client proof)
- `guides/new-client-onboarding.md` — P11.5 New Client Onboarding Guide (preset-based bootstrap workflow, infra overlay setup, seed pipeline, ellenőrzés)
- `phases/phase-11.5.md` — Phase 11.5 index (onboarding guide knowledge doc)
- `phases/phase-11.4.md` — Phase 11.4 Script Parameterization (-Client param, data-driven verify, image key walker)
- `phases/phase-11.3.md` — Phase 11.3 Platform + Infra Naming Refactor & Preset Scaffolding (@spektra/templates → @spektra/layouts, preset struktúra, 3 repo, audit F1–F4 javítva)
- `phases/phase-11.2.md` — Phase 11.2 Section Builder Delegation (generic registry dispatch, bc-* builders → kliens overlay, F1/P1 finding lezárva)
- `phases/phase-11.1.md` — Phase 11.1 Multi-Client Boundary Audit (4 repo audit, boundary matrix, 8 finding, bootstrap requirements)
- `concepts/wordpress-mapper-boundary.md` — WordPress Mapper Boundary (P8.1, kliens-oldali WP REST → SiteData mapper)
- `phases/phase-8.1.md` — Phase 8.1 index
- `phases/phase-8.5.md` — Phase 8.5 index
- `phases/phase-9.1.md` — Phase 9.1 index
- `phases/p9.1-runtime-cutover-baseline.md` — Runtime Cutover Baseline gate (phase-specific)

### Changed
- P12.3 strategy selected: Option C split tracks — P12.3a fixed_slots pilot planning, P12.3b BenettCar cpt_collection design
- P12.2 concept status `draft` → `accepted` promotálva; phase closeout dokumentálva; P12.3 planning note létrehozva
- `wp-integration-plan-v4.md` Appendix B — DR-003 státusz → ⚠️ Korrigálva (clarification hozzáadva: fixed field groups vs repeatable content scope); DR-015 hozzáadva (Repeatable Content Source Strategy)
- WordPress free baseline governance pontosítva: ACF Repeater (`type => 'repeater'`) ACF Pro feature, nem baseline-kompatibilis; `acf_add_local_field_group()` maga nem Pro-specifikus
- Gallery schema type consistency — `BcGalleryImage` (sp-benettcar) és `GalleryImage` (sp-exotica) `extends Media` platform típusra migrálva; lokális `src`/`alt` duplikáció megszüntetve
- Migrated knowledge structure to SPK v1.1.0 canonical model
  - Phase-first directories (`phase-8.5/`, `phase-9.1/`) → typed canonical dirs + `phases/` index
  - Type-view stub fájlok törölve (6 db)
  - Phase CHANGELOG-ok beolvasztva phase index fájlokba
  - Frontmatter: `phase:` → `phase_introduced:` + `canonical: true` (6 fájl)

> Runtime kód NEM módosult. Ez kizárólag dokumentációs és governance korrekció.

## [Pre-migration]

### Added
- `concepts/seed-pipeline.md` — Seed Pipeline koncepció (site.ts → WP szinkronizáció, 5-lépéses pipeline)
- `guardrails/parity-gate.md` — CMS Runtime Parity Gate guardrail (adapter switch előfeltétel)
- `guardrails/no-silent-fallback.md` — No Silent Fallback guardrail (WP adapter throw → null state → error slot)
- `implementation/image-sideload.md` — Image sideloading (external URL + lokális asset → WP media library)
- `implementation/error-ui-pattern.md` — Error UI Pattern (LandingTemplate error prop, branded kliens UI)
- `troubleshooting/group-key-normalization.md` — ACF group sub-field key normalizálás (suffix match)
