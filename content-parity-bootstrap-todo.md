# Phase 8.5 — Content Parity Bootstrap — TODO

> Workstream definíció: [content-parity-bootstrap.md](content-parity-bootstrap.md)
> Log: [content-parity-bootstrap-log.md](content-parity-bootstrap-log.md)

## P8.5.1 — Architecture Closure

- [x] content-parity-bootstrap.md létrehozása (ownership, boundary, guardrail, mini-phases)
- [x] content-parity-bootstrap-log.md scaffold
- [x] wp-integration-plan-v4.md §20 — Phase 8.5 sor hozzáadása
- [x] commit + push sp-docs (`41b6645`)

## P8.5.2 — Guardrail Definition

- [ ] Parity check spec véglegesítés (mező-szintű szabályok)
- [ ] Media exception szabály rögzítés (URL eltérés megengedett, alt+struktúra exact)
- [ ] commit + push sp-docs

## P8.5.3 — Tooling Design

- [ ] seed.json shape spec (flat ACF format, WP-CLI kompatibilis)
- [ ] ACF field mapping tábla (10 szekció — prefix + mezők)
- [ ] export-seed CLI interface design (--input, --mapping, --output)
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
