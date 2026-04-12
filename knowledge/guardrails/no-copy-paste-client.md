---
title: "No Copy-Paste Client — kliens létrehozás nem másolással"
status: draft
type: guardrail
phase_introduced: "12.1"
scope: platform
repos: [sp-platform, sp-clients]
tags: [guardrail, anti-pattern, copy-paste, onboarding, multi-client]
last_updated: 2026-04-12
canonical: true
---

# No Copy-Paste Client — kliens létrehozás nem másolással

## Summary (Simple)

Új Spektra kliens létrehozása NEM egy meglévő kliens fájljainak közvetlen másolásával történik. Az ismétlődő (Kategória A) fájlokat platform csomagokból kell importálni, a strukturálisan azonos (Kategória B) fájlokat scaffold/generátor hozza létre. Kézi másolás csak a valóban kliens-specifikus (Kategória D) fájloknál megengedett, és ott is a referencia implementáció (sp-benettcar) a kiindulás, nem egy random kliens.

## Technical Explanation

### Az anti-pattern

```
sp-benettcar/src/data/create-adapter.ts  ──copy──►  sp-exotica/src/data/create-adapter.ts
sp-benettcar/src/data/wp-mapper.ts       ──copy──►  sp-exotica/src/data/wp-mapper.ts
sp-benettcar/infra/seed/export-seed.ts   ──copy──►  sp-exotica/infra/seed/export-seed.ts
sp-benettcar/package.json                ──partial copy──►  sp-exotica/package.json  ← DRIFT
```

**Következmények:**
1. **Dependency amnesia** — A package.json nem másolódik teljesen: tsx, @types/node kimarad → build-time blocker
2. **Version drift** — ESLint, Vite, TypeScript verziók eltérnek a kliensenk között → reprodukálhatatlan buildek
3. **Patch propagation failure** — Ha a wp-mapper.ts-ben bug van, N kliensben kell külön javítani
4. **Implicit coupling** — A másoló fejlesztőnek tudnia kell, mit NEM kell módosítani (A kategória) vs mit kell (D kategória) — ez nincs dokumentálva a fájlban

### Az elvárt minta

```
@spektra/data (platform csomag)
  └── create-adapter.ts
  └── wp-mapper.ts
  └── normalize helpers (cleanOptional, cleanCta, ...)

sp-exotica/
  └── package.json: "@spektra/data": "workspace:*"
  └── src/data/normalize-site-data.ts: import { cleanOptional } from '@spektra/data'
```

### Döntési fa: másolni vagy importálni?

```
Fájl típusa?
├── 100% azonos minden kliensben (A)     → TILOS másolni → import from @spektra/*
├── Azonos struktúra, eltérő adat (B)    → NE másold → generáld vagy helper + kliens config
├── Package.json / config (C)            → NE másold → extends / shared base
└── Valóban kliens-specifikus (D)         → Másolható referenciából, MÓDOSÍTANDÓ
```

## Why It Exists

Az sp-exotica létrehozásakor (2. kliens) a benettcar fájljait copy-paste-tel vittük át. Ez három build-time blocker-t okozott (tsx hiányzik, @types/node hiányzik, URL-ek nem validáltak), és feltárta, hogy 6 fájl feleslegesen duplikálódik. A 3. kliensénél ez a módszer már nem skálázódik.

Az anti-pattern evidence a P12.1 boundary auditból:
- 6 fájl 100%-ban azonos (A kategória) → felesleges duplikáció
- 2 dependency kimaradt → B1, B3 blocker
- A patch (pl. wp-mapper.ts javítás) nem propagálódik automatikusan

## Implementation Notes

- **Amíg P1 elevation nincs kész:** a `guides/new-client-onboarding.md` a hivatalos út, nem a benettcar másolása
- **P1 után:** A kategória fájlok `@spektra/data` és `@spektra/seed` csomagokból jönnek
- **Ha mégis kell másolni:** csak D kategória fájlokat, és explicit checklist-tel (minden mezőt, prefixet, section type-ot módosítani)

## Boundaries and Guardrails

- **TILOS** Kategória A fájlt copy-paste-tel kliens repóba tenni (P1 elevation után)
- **TILOS** package.json-t másik kliens package.json-jéből másolni dependency-stül — a shared base config-ot kell extends-elni
- **Megengedett** D kategória fájlok másolása referencia implementációból, ha a fejlesztő végigmegy a módosítandó checklist-en
- **Owner:** platform team (sp-platform maintainerek döntik el, mi A és mi B kategória)

## Related Concepts

- [../concepts/platform-elevation.md](../concepts/platform-elevation.md) — A/B/C/D file classification és elevation path
- [../concepts/seed-pipeline.md](../concepts/seed-pipeline.md) — Seed Pipeline, ahol a copy-paste legtöbb A fájlja él
- [../troubleshooting/seed-pipeline-blockers.md](../troubleshooting/seed-pipeline-blockers.md) — Konkrét blocker-ek, amiket a copy-paste okozott
- [../guides/new-client-onboarding.md](../guides/new-client-onboarding.md) — A helyes onboarding út

## Open Questions

- Amíg P1 nem kész, a copy-paste „szükséges rossz" — van-e interim megoldás (pl. egy check script, ami összehasonlítja az A fájlokat)?
- Kellene-e CI check, ami FAIL-t ad, ha A kategória fájl eltér a platform verziótól?
