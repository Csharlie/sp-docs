---
title: ACF Group Sub-field Key Normalizálás
status: draft
type: troubleshooting
phase_introduced: "8.5"
canonical: true
repos: [sp-infra]
tags: [acf, group, dump, key-normalization, suffix-match, troubleshooting]
last_updated: 2026-04-11
---

# ACF Group Sub-field Key Normalizálás

## Summary (Simple)

Ha egy ACF group mezőt (pl. `bc_service_contact`) PHP-ból `get_field()`-del olvasunk, a visszaadott asszociatív tömb kulcsai **nem a rövid nevek** (pl. `title`), hanem a teljes ACF-prefixelt alakok (pl. `field_bc_service_contact_title`). A seed.json rövid kulcsokat használ. Ha a dump logika a raw kulcsokat írja wp-state.json-be, a parity check `undefined`-ot talál — hamis FAIL.

A megoldás: seed-shape-alapú kulcs-remapping suffix match-csel a `dump-acf.php` `normalize_dump_value()` függvényében.

## Technical Explanation

### A probléma

ACF group field-ek `get_field()` eredménye:

```php
// get_field('bc_service_contact', $post_id) visszaadja:
[
  'field_bc_service_contact_title'       => 'Egyeztetés és időpont',
  'field_bc_service_contact_phone'       => '+36 ...',
  'field_bc_service_contact_hours'       => 'H-P: 8:00-17:00',
  // ...
]
```

A seed.json-ben a group érték:

```json
{
  "bc_service_contact": {
    "title": "Egyeztetés és időpont",
    "phone": "+36 ...",
    "hours": "H-P: 8:00-17:00"
  }
}
```

A `verify-parity.ts` a seed kulcsait (`title`, `phone`, `hours`) keresi a wp-state.json-ben. Ha a dump a raw ACF kulcsokat írja, → `undefined` → FAIL.

### A megoldás: seed-shape suffix match

A `normalize_dump_value()` függvény opcionális `$seed_shape` paramétert kap. Ha a mező group típusú (asszociatív tömb), a seed shape kulcsait használja referenciaként:

```php
function normalize_dump_value($value, $seed_shape = null) {
    // ...
    if (is_array($value) && is_array($seed_shape) && !array_is_list($value)) {
        $remapped = [];
        foreach ($seed_shape as $seed_key => $seed_val) {
            // Suffix match: keresünk egy ACF kulcsot, ami _$seed_key-re végződik
            $found = false;
            foreach ($value as $acf_key => $acf_val) {
                if (str_ends_with($acf_key, '_' . $seed_key)) {
                    $remapped[$seed_key] = normalize_dump_value($acf_val);
                    $found = true;
                    break;
                }
            }
            if (!$found) {
                $remapped[$seed_key] = null;
            }
        }
        return $remapped;
    }
    // ...
}
```

Repeater esetén a seed shape első eleme (`$seed_shape[0]`) szolgál referenciának a sorok sub-field-jeihez.

### Érintett mezők

| ACF field | Típus | Sub-field-ek |
|---|---|---|
| `bc_service_contact` | group | title, description, phone, message_cta_text, message_cta_href, booking_note, hours, weekend_hours |
| `bc_contact_info` | group | phone, email, address |

## Why It Exists

A Phase 8.5 parity verification (P8.5.5) 53/55 eredményt adott. Az eredeti diagnózis — „az ACF field group regisztráció hiányzik" — **téves** volt. A regisztrációk (bc-service.php, bc-contact.php) végig léteztek. A valódi root cause a dump-acf.php volt, ami nem normalizálta a group sub-field kulcsokat.

Ez a hiba azért volt nehezen diagnosztizálható, mert:
1. Az ACF admin felületén a mezők helyesen jelentek meg
2. A `/spektra/v1/site` endpoint is helyes adatot adott (sections.php manuálisan kezeli a kulcsokat)
3. Csak a dump → verify path-on jelentkezett

## Implementation Notes

- **Fájl**: `sp-infra/seed/dump-acf.php` — `normalize_dump_value()` függvény
- **Hívás**: `normalize_dump_value($value, $seed['fields'][$key] ?? null)` — a seed mező értéke adja a shape-et
- **Suffix match logika**: `str_ends_with($acf_key, '_' . $seed_key)` — az ACF prefixet nem kell ismerni, elég a végződés
- **Repeater rekurzió**: `normalize_dump_value($row_val, $seed_shape[0] ?? null)` — az első seed sor a referencia
- **Commit**: `7884a0e` (sp-infra)

## Boundaries and Guardrails

- **Owner**: sp-infra (dump-acf.php)
- **Ez a normalizálás kizárólag a dump → verify path-ra vonatkozik** — a REST endpoint (sections.php) saját, manuális field mapping-et használ
- **NEM szabad**: a seed-shape-t hardcode-olni — mindig a seed.json-ból kell olvasni
- **NEM szabad**: a prefix-strippelést string replace-szel megoldani — a suffix match robusztusabb (nem függ a prefix konvenciótól)

## Related Concepts

- [p8.5-seed-pipeline.md](p8.5-seed-pipeline.md) — a pipeline, amelynek a dump lépése ezt használja
- `sp-infra/acf/bc-service.php` — bc_service_contact group regisztráció
- `sp-infra/acf/bc-contact.php` — bc_contact_info group regisztráció
- `sp-docs/postmortem-p8.5-content-parity.md` — „Korrigált diagnózis" szekció

## Open Questions / TODO

Nincs — a probléma megoldott, a pattern stabil.
