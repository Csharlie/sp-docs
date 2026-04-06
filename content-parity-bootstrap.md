# Phase 8.5 — Content Parity Bootstrap

> Interleaved workstream Phase 8 (closed) és Phase 9 (not started) között.
> A v4 roadmap fázisszámozás NEM változik — Phase 8.5 egy beszúrt mini-fázis.

## 1. Probléma

Phase 8.3 bevezette az adapter switcht (JSON → WP), de a seed pipeline (P10.4)
még 2 fázissal arrébb van. A WP adatbázisban dummy tartalom van (P6.4-ből),
miközben a `site.ts` valós, éles tartalom (~500 sor, 10 szekció, magyar szöveg).

Nincs garancia, hogy a kettő egyezik — és nincs gate, ami ezt kikényszerítené.

## 2. Pipeline

```
site.ts (SiteData)
  → export-seed.ts (Node script, kliens repo)
    → seed.json (generált, gitignored)
      → import-seed.sh (WP-CLI, infra repo)
        → WordPress DB (ACF field values)
```

## 3. Architecture Decisions

### 3.1. Canonical content source

`site.ts` (sp-benettcar) — ez az egyetlen igazság forrása. A WP adatbázis
tartalma ebből deriválódik, nem fordítva.

### 3.2. Boundary — repo ownership

| Fájl | Repo | Indoklás |
|---|---|---|
| `infra/seed/export-seed.ts` | sp-benettcar | kliens site.ts-t importálja, kliens mapping-et használja |
| `infra/seed/mapping.ts` | sp-benettcar | ACF field nevek = kliens-specifikus |
| `seed/import-seed.sh` | sp-infra | WP-CLI = infra művelet |
| `seed/verify-parity.ts` | sp-infra | parity check = platform-szintű gate |
| `seed/seed.json` | sp-infra | generált, gitignored |

**Miért nem sp-infra az export?**
Az export input (site.ts), a mapping, és az export logika mind kliens-specifikus.
Az sp-infra-ba helyezés cross-repo importot és kliensfüggő kódot vinne az infra rétegbe.

### 3.3. seed.json

- Generált fájl — **nem commitolható**
- Helye: `sp-infra/seed/seed.json`
- `.gitignore`-ban szerepel

### 3.4. sp-infra/seed/export-seed.ts scaffold

A P10.4-re készült scaffold rossz helyen van. P8.5.4-ben **törlendő** —
a canonical hely `sp-benettcar/infra/seed/export-seed.ts`.

## 4. Guardrail — CMS Runtime Parity Gate

> **Szabály**: Adapter runtime switch (JSON → WP) production cutover tilos addig,
> amíg a WP tartalom nem egyezik a site.ts tartalommal.
> A `verify-parity.ts` PASS eredménye előfeltétel.

### 4.1. Parity definíció

| Mező típus | Egyezési szint | Megjegyzés |
|---|---|---|
| Szöveg (title, description, body) | Exact match | Karakter-szintű egyezés |
| Navigation (label, href) | Exact match | Sorrend + tartalom |
| CTA (text, href) | Exact match | Ha site.ts-ben van, WP-ben is kell |
| Média (image src, alt) | Structural match | URL eltérhet (WP media library), alt text egyezik |
| Section order | Exact match | Szekció sorrend és type egyezik |

**Media exception**: Az image URL-ek eltérhetnek, mert a WP media library saját
URL-eket generál. Az `alt` text és a média struktúra (melyik szekcióban, hány kép)
viszont egyeznie kell.

## 5. Mini-fázisok

| # | Scope | Típus | Deliverable |
|---|---|---|---|
| P8.5.1 | Architecture Closure | Docs | Jelen dokumentum + log scaffold |
| P8.5.2 | Guardrail Definition | Docs | §4 véglegesítése, parity check spec |
| P8.5.3 | Tooling Design | Design + docs | seed.json shape, mapping spec |
| P8.5.4 | Minimal Implementation | Implementation | export-seed.ts, mapping.ts, import-seed.sh |
| P8.5.5 | Verification | Implementation | verify-parity.ts, PASS eredmény |
| P8.5.6 | Documentation | Docs | Postmortem, workflow standard, v4 plan update |

## 6. Phase 9 blokkolás

Phase 9 (E2E section bekötés) **NEM blokkolt** — párhuzamosan futhat.
De az adapter runtime switch production cutover TILOS a P8.5.5 PASS nélkül.

## 7. Új kliens onboarding checklist (Phase 11+ előkészítés)

1. `site.ts` → kliens tartalom megírása
2. `infra/seed/mapping.ts` → ACF field mapping
3. `infra/seed/export-seed.ts` → seed generálás
4. `import-seed.sh` → WP import
5. `verify-parity.ts` → PASS
6. Adapter switch engedélyezése
