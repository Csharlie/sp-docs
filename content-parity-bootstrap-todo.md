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
- [x] import-seed.sh interface design
- [x] image handling / media exception spec
- [ ] commit + push sp-docs

## P8.5.4 — Minimal Implementation

- [ ] mapping.ts (sp-benettcar/infra/seed/)
- [ ] export-seed.ts (sp-benettcar/infra/seed/)
- [ ] sp-infra/seed/export-seed.ts scaffold törlés
- [ ] import-seed.sh (sp-infra/seed/)
- [ ] seed.json .gitignore (sp-infra/seed/)
- [ ] commit + push sp-benettcar
- [ ] commit + push sp-infra

## P8.5.5 — Verification

- [ ] verify-parity.ts (sp-infra/seed/)
- [ ] seed generálás + WP import futtatás
- [ ] parity check PASS (mind 10 szekció)
- [ ] commit + push sp-infra

## P8.5.6 — Documentation

- [ ] postmortem-p8.5-sequencing-slip.md (sp-docs)
- [ ] workflow standard + onboarding checklist véglegesítés
- [ ] log frissítés (content-parity-bootstrap-log.md — all entries)
- [ ] commit + push sp-docs
