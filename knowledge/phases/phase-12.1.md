# Phase 12.1 — Second Client Validation (sp-exotica)

## Purpose

A Phase 12.1 az sp-exotica (Exotica by Bella) kliens létrehozása és seed pipeline implementálás — az első alkalom, hogy a platform multi-client képessége éles körülmények között validálódik. A Phase 11.1–11.5 audit és onboarding guide kiadása után ez a „proof by execution" fázis.

---

## Kontextus

- **Kliens:** Exotica by Bella — egzotikus állatkereskedés
- **Prefix:** `eb`
- **Szekciók:** hero, about, products, animals, gallery, contact (6 db)
- **Pipeline eredmény:** PASS — 39/39 parity, 20/20 endpoint
- **Módszer:** sp-benettcar fájlok manuális másolása + módosítás (copy-paste anti-pattern)

---

## Concepts

- [Platform Elevation](../concepts/platform-elevation.md) — A/B/C/D file classification és elevation path (kanonikus)

## Guardrails

- [No Copy-Paste Client](../guardrails/no-copy-paste-client.md) — Kliens létrehozás nem másolással (kanonikus)

## Troubleshooting

- [Seed Pipeline Blockers](../troubleshooting/seed-pipeline-blockers.md) — 5 blocker taxonomy: B1–B5 (kanonikus)

## Implementation

_(Nincs új kanonikus implementation doc ebben a fázisban.)_

---

## Phase-specific notes

### Fő tanulságok

1. **Copy-paste nem skálázódik** — 6 fájl 100%-ban azonos, feleslegesen duplikálva
2. **Dependency amnesia** — tsx és @types/node kimaradt a másolásból → build blocker
3. **Unsplash URL-ek instabilak** — 3 fotó, 5 előfordulás, 3 pipeline futtatás kellett a PASS-hoz
4. **ACF update_field false** — expected behavior, de megtévesztő log
5. **A pipeline maga generikus** — sp-infra scriptek változtatás nélkül működtek az exoticára

### Érintett fájlok az sp-exotica-ban

| Fájl | Kategória | Megjegyzés |
|------|-----------|------------|
| `infra/seed/mapping.ts` | B | 6 szekció, 37 mező mapping |
| `infra/seed/export-seed.ts` | A | Benettcar-ból copy-paste, 100% azonos |
| `infra/seed/asset-loader.mjs` | A | Benettcar-ból copy-paste, 100% azonos |
| `infra/seed/register-asset-hooks.mjs` | A | Benettcar-ból copy-paste, 100% azonos |
| `infra/seed/tsconfig.json` | A | Benettcar-ból copy-paste, 100% azonos |
| `src/data/create-adapter.ts` | A | Benettcar-ból copy-paste, 100% azonos |
| `src/data/wp-mapper.ts` | A | Benettcar-ból copy-paste, 100% azonos |

### Forrás dokumentáció (kliens repóban)

- `sp-clients/sp-exotica/docs/HANDOFF-seed-pipeline.md` — Seed pipeline handoff
- `sp-clients/sp-exotica/docs/seed-pipeline-blockers.md` — Részletes blocker dokumentáció
- `sp-docs/platform-elevation-plan.md` — Teljes elevation plan (I. + II. rész)
