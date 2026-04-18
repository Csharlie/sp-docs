# Knowledge Changelog

Minden knowledge dokumentum változás naplója. Keep a Changelog formátum.

## [Unreleased]

### Added
- `concepts/repeatable-content-source-strategy.md` — P12.2 Repeatable Content Source Strategy: 3 source strategy (cpt_collection / fixed_slots / acf_repeater_optional), SiteData stability rule, builder abstraction boundary, 13 repeater classification
- `phases/phase-12.2.md` — Phase 12.2 index: Repeatable Content Baseline Correction (governance mismatch DR-003, transition rules, classification table, migration plan)
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
