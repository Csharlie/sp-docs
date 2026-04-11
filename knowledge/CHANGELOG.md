# Knowledge Changelog

Minden knowledge dokumentum változás naplója. Keep a Changelog formátum.

## [Unreleased]

### Added
- `phases/phase-11.1.md` — Phase 11.1 Multi-Client Boundary Audit (4 repo audit, boundary matrix, 8 finding, bootstrap requirements)
- `concepts/wordpress-mapper-boundary.md` — WordPress Mapper Boundary (P8.1, kliens-oldali WP REST → SiteData mapper)
- `phases/phase-8.1.md` — Phase 8.1 index

### Changed
- Migrated knowledge structure to SPK v1.1.0 canonical model
  - Phase-first directories (`phase-8.5/`, `phase-9.1/`) → typed canonical dirs + `phases/` index
  - Type-view stub fájlok törölve (6 db)
  - Phase CHANGELOG-ok beolvasztva phase index fájlokba
  - Frontmatter: `phase:` → `phase_introduced:` + `canonical: true` (6 fájl)

### Added
- `phases/phase-8.5.md` — Phase 8.5 index
- `phases/phase-9.1.md` — Phase 9.1 index
- `phases/p9.1-runtime-cutover-baseline.md` — Runtime Cutover Baseline gate (phase-specific)

## [Pre-migration]

### Added
- `concepts/seed-pipeline.md` — Seed Pipeline koncepció (site.ts → WP szinkronizáció, 5-lépéses pipeline)
- `guardrails/parity-gate.md` — CMS Runtime Parity Gate guardrail (adapter switch előfeltétel)
- `guardrails/no-silent-fallback.md` — No Silent Fallback guardrail (WP adapter throw → null state → error slot)
- `implementation/image-sideload.md` — Image sideloading (external URL + lokális asset → WP media library)
- `implementation/error-ui-pattern.md` — Error UI Pattern (LandingTemplate error prop, branded kliens UI)
- `troubleshooting/group-key-normalization.md` — ACF group sub-field key normalizálás (suffix match)
