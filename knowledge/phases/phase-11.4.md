# Phase 11.4 — Script Parameterization

## Purpose

P11.4 a seed pipeline és verify endpoint szkripteket parametrizálja, hogy tetszőleges kliens slug-gal futtathassák — nem csak a hardcoded `benettcar`-ral.

---

## Érintett fájlok

### sp-infra/seed/seed-pipeline.ps1

| Változás | Részlet |
|----------|---------|
| Új paraméter | `-Client` (default: `'benettcar'`) |
| Dinamikus path | `$ClientDir = Join-Path $SpRoot "sp-clients\sp-$Client"` |
| Dinamikus path | `$WpRuntime = Join-Path $SpRoot ".local\wp-runtimes\$Client"` |
| Docs | `.PARAMETER Client` + `.EXAMPLE` blokkok frissítve |

### sp-infra/seed/verify-endpoint.php

| Változás | Részlet |
|----------|---------|
| Hardcoded törlés | ~35 soros `switch($id)` blokk (bc-hero, bc-brand, …) eltávolítva |
| Új logika | Data-driven image field walker: `$image_keys` tömb |
| Keys | `['image', 'backgroundImage', 'logo', 'src', 'photo', 'avatar', 'cover']` |
| Működés | Top-level mezők + nested array mezők bejárása, `false` / missing → WARNING |

---

## Tervezési döntések

1. **Default paraméter** — `-Client 'benettcar'` default megmarad, így a meglévő workflow nem törik
2. **Data-driven walker** — A `verify-endpoint.php` nem igényli a section-specifikus field list-et, hanem konvencióalapú key-eket keres. Ez eliminál minden kliens-specifikus PHP kódot az infra repóból.
3. **Image key konvenció** — Bármely mező, aminek a neve megegyezik az `$image_keys` lista egy elemével, ellenőrzésre kerül. Ez kiterjeszthető lista.

---

## Verifikáció

- PHP lint PASS (`php -l verify-endpoint.php`)
- PowerShell syntax PASS (`[System.Management.Automation.Language.Parser]::ParseFile()`)
- Seed pipeline futtatható: `.\seed-pipeline.ps1 -Client benettcar`

---

## Commit

| Repo | Hash | Message |
|------|------|---------|
| sp-infra | `c8090ac` | `feat(seed): parameterize seed-pipeline + data-driven verify` |
