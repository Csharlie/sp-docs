---
title: Seed Pipeline Blocker Taxonomy
status: draft
type: troubleshooting
phase_introduced: "12.1"
scope: cross-repo
repos: [sp-clients, sp-infra]
tags: [seed, pipeline, blocker, troubleshooting, dependency, unsplash, acf]
last_updated: 2026-04-12
canonical: true
---

# Seed Pipeline Blocker Taxonomy

## Summary (Simple)

Az sp-exotica seed pipeline implementálásakor 5 blocker típust azonosítottunk. Ezek kategorizálhatók és megelőzhetők. A dokumentum a root cause-t, a megoldást és a prevention-t tartalmazza minden blocker-hez — így a következő kliensnél nem kell újra felfedezni.

## Blocker-ek

### B1: Hiányzó devDependency — tsx

| | |
|---|---|
| **Szint** | Build-time, blocking |
| **Tünet** | `pnpm seed:export` → „tsx: command not found" |
| **Root cause** | Seed fájlok a benettcar-ból másolva, de a `package.json` devDependencies nem — a tsx csomag kimaradt |
| **Megoldás** | `pnpm add -D tsx` |
| **Prevention** | Scaffold generátor, vagy A kategória fájlok platform csomagba emelése (P1 elevation) |
| **Érintett scope** | Minden jövőbeli kliens, ahol seed pipeline-t használunk |

### B2: Unsplash URL instabilitás

| | |
|---|---|
| **Szint** | External resource, soft failure |
| **Tünet** | `import-seed.php` → `media_sideload_image()` → HTTP 404; endpoint smoke → BROKEN |
| **Root cause** | Unsplash CDN nem garantálja URL állandóságot — fotók törlődhetnek, licenc visszavonható |
| **Hatás** | Sideload skip-eli a hibás URL-t, WP-ben null az attachment, endpoint smoke BROKEN |
| **Megoldás** | URL csere validált fotókra + `Invoke-WebRequest -Method Head` pre-flight check |
| **Prevention** | Pre-flight URL validation a pipeline-ba: export után, import előtt HTTP HEAD minden image URL-re |
| **Recommendation** | Beépítendő `seed-pipeline.ps1` lépés 1 és 2 közé |

### B3: Hiányzó devDependency — @types/node

| | |
|---|---|
| **Szint** | DX (Developer Experience), non-blocking for runtime |
| **Tünet** | VSCode: „Cannot find type definition file for 'node'" (piros tsconfig.json) |
| **Root cause** | Azonos mint B1 — `infra/seed/tsconfig.json` referálja `"types": ["node"]`, de a csomag nincs devDependencies-ben |
| **Specifikus** | Csak VSCode diagnostics-ban jelenik meg; a tsx futtatás ettől működik (saját Node típusai vannak) |
| **Megoldás** | `pnpm add -D @types/node` |
| **Prevention** | Azonos mint B1 — scaffold generátor vagy shared config |

### B4: ACF update_field() false ismételt futtatásnál

| | |
|---|---|
| **Szint** | Expected behavior, NEM bug |
| **Tünet** | Import log: `[FAIL] update_field returned false` (~35 mező, második futtatástól) |
| **Root cause** | WordPress `update_metadata()` `false`-t ad, ha az új érték megegyezik a meglévővel — WP core viselkedés 2.x óta |
| **Hatás** | Megtévesztő log — a fejlesztő azt hiheti, elromlott valami |
| **Parity impact** | Nincs — a dump az aktuális WP állapotot nézi, az nem változott → PASS |
| **Recommendation** | Az `import-seed.php`-ban: `update_field() === false` && `get_field() === $new_value` → `[SKIP/UNCHANGED]`, nem `[FAIL]` |

### B5: Nem validált csere-URL

| | |
|---|---|
| **Szint** | Process gap, emberi hiba |
| **Tünet** | Csere-URL is 404 → +1 teljes pipeline futtatás |
| **Root cause** | URL cserénél nem futott `Invoke-WebRequest -Method Head` teszt a jelölt URL-re |
| **Hatás** | +1 pipeline futtatás (~90 mp), +2 felesleges sideload kísérlet, attachment maradvány WP-ben |
| **Prevention** | Mindig HEAD request validáció csere-URL beírása előtt; vagy pipeline-ba beépített pre-flight check (lásd B2) |

## Blocker összefoglaló mátrix

| # | Típus | Severity | Megelőzhető? | Prevention eszköz |
|---|-------|----------|--------------|-------------------|
| B1 | Missing dependency | Blocking | Igen | Scaffold / P1 elevation |
| B2 | External resource | Soft failure | Részben | Pre-flight URL check |
| B3 | Missing dependency | DX only | Igen | Scaffold / P1 elevation |
| B4 | Expected behavior | False alarm | Igen | Jobb log üzenet |
| B5 | Process gap | Extra idő | Igen | Pre-flight URL check |

## Pipeline futtatások száma az exotica PASS-ig

```
1. futtatás → 5 broken image (B2)
2. futtatás → 3 broken image (B5 — csere-URL is 404)
3. futtatás → 0 broken → PASS (39/39 parity, 20/20 endpoint)
```

Optimális: 1 futtatás. A B2 pre-flight check bevezetése után ez elérhető.

## Why It Exists

Az sp-exotica a második Spektra kliens. A seed pipeline implementálása során 5 blocker-t találtunk, amelyek közül 3 (B1, B3, B5) megelőzhető lett volna. Dokumentáljuk, hogy a harmadik kliensnél ne forduljanak elő újra.

## Implementation Notes

- **B1+B3 javítás:** A `guides/new-client-onboarding.md`-ben explicit devDependency lista van — de a copy-paste workflow nem a guide-ot követi
- **B2+B5 javítás:** Pre-flight URL validation script koncepció:
  ```powershell
  $urls = (Get-Content seed.json | ConvertFrom-Json).fields.PSObject.Properties |
    Where-Object { $_.Value -match '^https?://' } |
    ForEach-Object { $_.Value }
  $urls | ForEach-Object {
    try { Invoke-WebRequest -Uri $_ -Method Head -TimeoutSec 5 -UseBasicParsing | Out-Null; "[OK] $_" }
    catch { "[FAIL] $_" }
  }
  ```
- **B4 javítás:** `import-seed.php` log módosítás — low effort, high DX impact

## Boundaries and Guardrails

- Ez a taxonomy az sp-exotica tapasztalatából származik, de **minden kliens seed pipeline-jára vonatkozik**
- B4 nem implementálandó javításként ebben a fázisban — csak awareness szükséges
- A pre-flight URL check (B2/B5) P1 elevation scope-ba tartozik

## Related Concepts

- [../concepts/seed-pipeline.md](../concepts/seed-pipeline.md) — Seed Pipeline koncepció
- [../concepts/platform-elevation.md](../concepts/platform-elevation.md) — Platform Elevation (B1/B3 prevention: P1 elevation)
- [../guardrails/no-copy-paste-client.md](../guardrails/no-copy-paste-client.md) — No Copy-Paste Client guardrail
- `sp-clients/sp-exotica/docs/seed-pipeline-blockers.md` — Eredeti részletes blocker dokumentáció

## Open Questions

- Pre-flight URL check: a `seed-pipeline.ps1`-be vagy az `export-seed.ts`-be kerüljön?
- B4 log javítás: van-e más ACF funkció, ami hasonlóan megtévesztő return value-val dolgozik?
