# Phase 8.5 — Content Parity Bootstrap — TODO

> Workstream definíció: [content-parity-bootstrap.md](content-parity-bootstrap.md)
> Log: [content-parity-bootstrap-log.md](content-parity-bootstrap-log.md)

## P8.5.1 — Architecture Closure

- [x] content-parity-bootstrap.md létrehozása (ownership, boundary, guardrail, mini-phases)
- [x] content-parity-bootstrap-log.md scaffold
- [x] wp-integration-plan-v4.md §20 — Phase 8.5 sor hozzáadása
- [x] commit + push sp-docs (`41b6645`)

## P8.5.2 — Guardrail Definition

- [x] Parity check spec véglegesítés (v4 §13.2 ownership alignment)
- [x] Media exception szabály rögzítés (content-parity-bootstrap.md §4.1)
- [x] seed.json gitignore hozzáadása (sp-infra .gitignore)
- [x] seed README boundary alignment (sp-infra seed/README.md)
- [x] P10.4 → P8.5 alias note (wp-integration-plan-v4.md §20)
- [x] commit + push sp-docs (`235a6b1`)
- [x] commit + push sp-infra (`38f095d`)

## P8.5.3 — Tooling Design

- [x] seed.json shape spec (flat ACF format, WP-CLI kompatibilis)
- [x] ACF field mapping tábla (10 szekció — prefix + mezők)
- [x] export-seed CLI interface design (--output, --dry-run, --verbose)
- [x] import-seed.php interface design (ACF API)
- [x] image handling / media exception spec
- [x] commit + push sp-docs (`42e10bb`)

## P8.5.4 — Minimal Implementation

- [x] mapping.ts (sp-benettcar/infra/seed/)
- [x] export-seed.ts (sp-benettcar/infra/seed/)
- [x] sp-infra/seed/export-seed.ts scaffold törlés
- [x] import-seed.php (sp-infra/seed/) — ACF-aware, wp eval-file
- [x] ACF alt fields: bc-hero, bc-about, bc-team (sp-benettcar)
- [x] media-helper.php alt_override + sections.php alt pass-through (sp-infra)
- [x] seed.json .gitignore (sp-infra — already in P8.5.2)
- [x] commit + push sp-benettcar (`c4ed7f6`)
- [x] commit + push sp-infra (`7aa1964`)
- [x] commit + push sp-docs meta (`9ff505f`)

## P8.5.5 — Verification

- [x] verify-parity.ts (sp-infra/seed/) — `4006047`
- [x] dump-acf.php (sp-infra/seed/) — `4006047`
- [x] dump-acf.php image round-trip shape fix — `ef237c0`
- [x] seed.json generálás (53 mező, 10 szekció)
- [x] @types/node + infra/seed/tsconfig.json — `d3697a7`
- [x] seed-pipeline.ps1 orchestrátor — `60555fa`
- [x] WP import + dump (éles pipeline futtatás)
- [x] parity check: **55/55 PASS** (korrigált — lásd P9.2 javítás)
- [x] commit + push sp-infra (`4006047`, `ef237c0`, `60555fa`)
- [x] commit + push sp-benettcar (`6d94056`, `d3697a7`)

## P8.5.6 — Documentation

- [x] postmortem-p8.5-content-parity.md (sp-docs) — honest close + workflow standard
- [x] log frissítés: #14, #15 entry + close header
- [x] todo frissítés: P8.5.5 + P8.5.6 lezárás
- [ ] commit + push sp-docs

---

## Phase 8.5 — CLOSED

**Eredmény:** 55/55 verified parity + 18/18 endpoint smoke, működő 5-lépéses pipeline.

> **Korrekció (P9.2):** Az eredeti 53/55 diagnózis téves volt — a group mezők ACF
> regisztrációja nem hiányzott (bc-service.php, bc-contact.php definíciók végig léteztek).
> A FAIL oka: dump-acf.php raw ACF key prefix a group sub-field-eken.
> Fix: seed-shape-alapú kulcs-remapping a normalize_dump_value()-ban.
> A brand logókat is sideload-olja a pipeline (lokális fájlokból WP media library-be).
