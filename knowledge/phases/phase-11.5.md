# Phase 11.5 — New Client Onboarding Guide

## Purpose

P11.5 egy standalone onboarding guide létrehozása, ami végigvezeti az új kliens projekt teljes bootstrapját a preset-alapú rendszerrel. Ez a P11.1–P11.4 eredményeire épít.

---

## Létrehozott dokumentum

`sp-docs/knowledge/guides/new-client-onboarding.md`

---

## A guide tartalma

| Szekció | Leírás |
|---------|--------|
| Előfeltételek | Node, pnpm, PHP, WP, ACF, workspace struktúra |
| Preset kiválasztás | baseline / landingline / shopline — melyiket válasszuk |
| Kliens projekt | Mappa struktúra, package.json, workspace link |
| Infra overlay | config.php, ACF field-groups, section definíciók, env, builders |
| WP runtime | Runtime mappa, link-plugin.ps1, link-overlay.ps1, ACF aktiválás |
| Seed pipeline | seed-pipeline.ps1 -Client, dry-run, verbose |
| Frontend | pnpm install, pnpm dev |
| Ellenőrzés | Build, test, REST endpoint, CORS, image fields |
| Referencia | sp-benettcar teljes struktúra |
| Hibakeresés | Gyakori hibák + megoldások |

---

## Tervezési döntések

1. **Preset-based, nem starter-based** — Az `apps/playground` (korábban `apps/starter`) NEM az onboarding forrás. A presets/ struktúra az.
2. **Generic, nem benettcar-specifikus** — `{slug}` placeholderek, nem hardcoded benettcar referenciák.
3. **Self-contained** — A guide önállóan követhető, de hivatkozik a részletes docs-ra (cors-guide, smoke-test-guide, stb.)
4. **Futtatható parancsok** — Minden lépéshez PowerShell parancs, copy-paste ready.
