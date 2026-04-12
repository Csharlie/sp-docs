---
title: Platform Elevation — kliens-duplikáció felszámolása
status: draft
type: concept
phase_introduced: "12.1"
scope: platform
repos: [sp-platform, sp-clients]
tags: [elevation, deduplication, shared-packages, multi-client, architecture]
last_updated: 2026-04-12
canonical: true
---

# Platform Elevation — kliens-duplikáció felszámolása

## Summary (Simple)

Platform Elevation: azonosítani, mely kliens-oldali fájlok azonosak minden kliensben, és ezeket közös platform csomagokba emelni. Így az új kliens létrehozásakor importál, nem másol. A koncepció az sp-exotica (második kliens) létrehozásakor kristályosodott ki, amikor a benettcar fájljainak részletes összehasonlítása feltárta a négy fájlkategóriát.

## Technical Explanation

### Fájl-osztályozási modell (A/B/C/D)

A kliens repó fájljai négy kategóriába sorolhatók a platform-level reuse szempontjából:

| Kategória | Definíció | Elevation stratégia |
|-----------|-----------|---------------------|
| **A — 100% azonos** | Byte-szinten megegyezik minden kliensben | Azonnali: kiemelés shared csomagba |
| **B — Azonos struktúra, eltérő tartalom** | Pattern fix, adat változó (prefix, szekció-lista) | Középtávú: generálás schema-ból vagy helper+config |
| **C — Package.json drift** | Dependency-k eltérnek verziókban vagy hiányoznak | Shared base config vagy scaffold generátor |
| **D — Valóban kliens-specifikus** | Tartalom, design, üzleti logika | Marad a kliens repóban |

### Kategória A — azonnali deduplikáció jelöltjei

| Fájl | Jelenlegi hely | Javasolt hely |
|------|---------------|---------------|
| `src/data/create-adapter.ts` | `sp-{slug}/src/data/` | `@spektra/data` |
| `src/data/wp-mapper.ts` | `sp-{slug}/src/data/` | `@spektra/data` |
| `infra/seed/export-seed.ts` | `sp-{slug}/infra/seed/` | `@spektra/seed` |
| `infra/seed/asset-loader.mjs` | `sp-{slug}/infra/seed/` | `@spektra/seed` |
| `infra/seed/register-asset-hooks.mjs` | `sp-{slug}/infra/seed/` | `@spektra/seed` |
| `infra/seed/tsconfig.json` | `sp-{slug}/infra/seed/` | `@spektra/seed` (base → extends) |

**Különbségek a két kliensben:** kizárólag `@package` JSDoc comment — funkcionális eltérés nulla.

### Kategória B — generálható fájlok

| Fájl | Mi a kliens-specifikus | Közös pattern |
|------|------------------------|---------------|
| `infra/seed/mapping.ts` | Szekció lista, ACF prefix, extract függvények | FieldMapping/SectionMapping típusok, str/bool/extractCta/extractMedia helperek |
| `src/data/normalize-site-data.ts` | switch-case szekciónként, normalizer függvények | cleanOptional, cleanCta, isDefined, hasRenderableHero helperek |
| `infra/acf/builders.php` | Builder függvények szekciónként | `spektra_build_{prefix}_{section}()` pattern, `spektra_get_field()` hívás |
| `infra/acf/sections/*.php` | Prefix stringek, mező nevek | ACF `acf_add_local_field_group()` struktúra, field type patterns |

**Közös minta:** Mind a négy fájltípusnál a szekció-lista + mező-lista a paraméter. A pattern fix, a data változó.

### Kategória C — dependency drift

Kliens copy-paste során dependency-k kimaradnak (tsx, @types/node) vagy verziókülönbségek keletkeznek (ESLint, Vite major version). Ez nem fájlszintű duplikáció, hanem konfigurációs drift.

### Kategória D — kliens-specifikus (nem emelendő)

- `site.ts` — tartalom (szöveg, URL-ek, meta)
- `types.ts` — kliens section data típusok
- `sections/*.tsx` — React komponensek (design, layout)
- `theme/` — Tailwind konfig, színek, betűtípusok
- `.env` — kliens-specifikus WP URL, port
- `mapping.ts` extract függvények (a fájl struktúrája B, de az extract logika D)

### Elevation útvonal (P1/P2/P3)

| Prioritás | Leírás | Effort | Hatás |
|-----------|--------|--------|-------|
| **P1 (azonnali)** | Kategória A → `@spektra/data` + `@spektra/seed` | Alacsony — import path cseréje | Megszűnik a duplikáció, platform javítás automatikusan eljut |
| **P2 (középtávú)** | ACF field group generator YAML schema-ból | Közepes — schema formátum definiálás | Kategória B PHP fájlok ~100% generálhatók |
| **P3 (hosszútávú)** | `spektra create-client` CLI scaffold | Magas — CLI tooling | Teljes kliens generálás parancssorból |

## Why It Exists

Az sp-exotica az első kliens, amit az sp-benettcar alapján hoztunk létre. A másolási folyamat során kiderült:
- 6 fájl 100%-ban azonos (Kategória A) — ezek feleslegesen duplikálódnak
- 4 fájltípus azonos mintát követ (Kategória B) — manuális copy-paste helyett generálhatók lennének
- 2 dependency kimaradt a másolásból (Kategória C) — build-time blocker lett belőle
- A platform javítás NEM jut el automatikusan a kliens másolatokba

Ez a probléma a harmadik kliensénél már fenntarthatatlan: N kliens × 6 azonos fájl = N másolat, amelyeket szinkronban kell tartani.

## Implementation Notes

- A P1 elevation breaking change: a kliens `import` path-ek módosulnak (pl. `./data/create-adapter` → `@spektra/data`)
- A `package.json`-ba workspace dependency kell: `"@spektra/data": "workspace:*"`
- A seed tsconfig.json `extends` pattern-nel oldható meg: kliens `extends: "@spektra/seed/tsconfig.seed.json"`
- A B kategória mapping.ts résznél a helper-ek emelhetők, de az extract függvények kliens-specifikusak maradnak

## Boundaries and Guardrails

- **Kategória A fájloknál:** ha bármelyik kliens speciális logikát igényelne, az nem A kategória — vissza kell sorolni B-be
- **Ne generáld automatikusan a D kategóriát** — az tartalom és design döntés, nem kód
- **Hibrid fájlok (B):** a közös helper-ek emelhetők, de a kliens-specifikus rész maradjon a kliensben
- **A teljes elevation plan referencia:** `sp-docs/platform-elevation-plan.md`

## Related Concepts

- [seed-pipeline.md](seed-pipeline.md) — Seed Pipeline koncepció, ami a B kategória fő use case-e
- [../guardrails/no-copy-paste-client.md](../guardrails/no-copy-paste-client.md) — Copy-paste kliens anti-pattern
- [../guides/new-client-onboarding.md](../guides/new-client-onboarding.md) — Kliens onboarding guide (P11.5)
- `sp-docs/platform-elevation-plan.md` — Teljes elevation plan (I. + II. rész)

## Open Questions

- P1 implementáció mikor? A jelenlegi 2 kliens mellett még kezelhető, de a 3. kliens előtt mandatory
- P2 schema formátum: YAML vs JSON vs TypeScript config object?
- P3 CLI: standalone tool vagy pnpm script?
