---
title: WordPress Mapper Boundary
status: draft
type: concept
phase_introduced: "8.1"
canonical: true
scope: client
repos: [sp-benettcar]
tags: [mapper, boundary, wordpress, client-layer, section-aware, sitedata]
last_updated: 2026-04-11
---

# WordPress Mapper Boundary

## Summary (Simple)

A WordPress Mapper Boundary az a kliens-oldali réteg, amely a nyers WP REST API választ (`/spektra/v1/site`) átalakítja a frontend által várt `SiteData` formátumra. Ez az egyetlen hely, ahol a WordPress payload struktúrája ismert — sem a React komponensekben, sem a platform csomagokban, sem az App.tsx-ben nem jelenhet meg WP-specifikus logika.

## Technical Explanation

### Architektúra pozíció

```
sp-infra (WP plugin)
    ↓
WP REST payload (raw JSON)
    ↓
sp-benettcar/src/data/wp-mapper.ts   ← BOUNDARY
    ↓
SiteData (platform contract)
    ↓
React sections
```

### Publikus API

Egyetlen entry point:

```ts
mapWordPressSiteData(raw: unknown): SiteData
```

### Belső struktúra

A mapper nem generikus deep-mapper, hanem **section-aware boundary layer**:

```
mapWordPressSiteData()
  ├── mapSite()
  ├── mapNavigation()
  ├── mapPages()
  │     └── mapPage()
  │           └── mapSections()
  │                 └── mapSection()
  │                       ├── mapBcHero()
  │                       ├── mapBcBrand()
  │                       ├── mapBcGallery()
  │                       ├── mapBcServices()
  │                       ├── mapBcAbout()
  │                       ├── mapBcTeam()
  │                       ├── mapBcAssistance()
  │                       ├── mapBcContact()
  │                       └── mapBcMap()
  │
  └── (ismeretlen section type → skip, fail-soft)
```

### Hibakezelés

- **Top-level struktúra hibás** (hiányzik `site`, `navigation`, `pages`) → **throw**. Nem ad vissza üres SiteData-t.
- **Ismeretlen vagy malformed section** → **skip** (fail-soft). Nem töri el az egész oldalt.
- **Optional mezők**: `null → undefined` átalakítás a React kompatiblitás érdekében.

### Speciális mező kezelés

| Mező | Típus | Megjegyzés |
|------|-------|------------|
| `bc-hero.backgroundImage` | Media | Teljes Media objektum |
| `bc-about.image` | Media | Teljes Media objektum |
| `bc-team.members[].image` | Media | Teljes Media objektum |
| `bc-brand.brands[].logo` | string | Szándékosan string, nem Media |
| `bc-gallery.images[].src` | string | Szándékosan string, nem Media |

## Why It Exists

P8.1 előtt nem volt explicit hely, ahol a WordPress payload átalakítása történt. A mapper nélkül a WP-specifikus logika szétszóródna a komponensekben, ami:
- Lehetetlenné tenné az adapter switch-et (JSON ↔ WP)
- Összemosná a platform/client/infra határokat
- Tesztelhetetlen lenne

A mapper létrehozása biztosítja, hogy a WP payload struktúrát **pontosan egy fájl** ismeri a kliens repóban.

## Implementation Notes

- **Fájl**: `sp-benettcar/src/data/wp-mapper.ts`
- **Commit**: `48d8596` (P8.1, sp-benettcar)
- **Verifikáció**: 23/23 mapping assertion PASS az eredeti implementációban
- **Későbbi hardening** (P8.x): `emptySiteData()` fallback eltávolítva, top-level throw bevezetve
- Helper függvények: `asString()`, `maybeMedia()`, `maybeCta()`, `optionalString()`, `isRecord()`

## Boundaries and Guardrails

- **Owner**: kliens réteg (`sp-benettcar`)
- **NEM szabad**: WP-specifikus logikát React komponensekben tartani
- **NEM szabad**: inline mapping-et App.tsx-ben végezni
- **NEM szabad**: platform csomagokat módosítani a mapper miatt
- **NEM szabad**: infra csomagokat módosítani a mapper miatt
- **Minden kliens repó** saját mappert implementál — a mapper kliens-specifikus, nem megosztott platform logika
- A mapper kizárólag a `SiteData` platform contractot célozza — ha a contract változik, a mapper frissül

## Related Concepts

- [no-silent-fallback.md](../guardrails/no-silent-fallback.md) — a mapper throw viselkedését erősíti meg runtime szinten
- [seed-pipeline.md](../concepts/seed-pipeline.md) — a seed pipeline az inverze: SiteData → WP; a mapper: WP → SiteData

## Open Questions

- Generikus mapper-generator a jövőben? Jelenleg nincs rá igény — a section-aware megközelítés explicitebbé teszi a mappingot.
