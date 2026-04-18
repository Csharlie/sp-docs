---
title: "P12.4 — eb-about.values: ACF Repeater → Textarea+Split Migration"
status: executed
type: implementation-prompt
phase: "12.4"
canonical: true
repos: [sp-exotica]
tags: [fixed_slots, migration, acf, textarea, repeater, eb-about]
last_updated: 2026-04-18
planning_ref: phases/phase-12.3.md#P12.3a
---

# P12.4 — eb-about.values: ACF Repeater → Textarea+Split Migration

## Summary

Az `eb-about` szekció `values` mezőjének migrálása ACF Repeater-ről ACF Free textarea mezőre. A PHP builder `spektra_split_textarea()` helperrel bontja sorokra a textarea tartalmat. A SiteData output (`values: string[]`) változatlan marad.

**Scope:** Kizárólag sp-exotica repó. sp-infra NEM módosul.

---

## Context

- **Phase:** P12.4 (first runtime migration, P12.3a pilot)
- **Source strategy:** `fixed_slots` (bounded kis tartalom, ACF Free-kompatibilis)
- **Planning:** `phases/phase-12.3.md` P12.3a szekció — minden planning output lezárt és READY
- **Guardrail:** SiteData shape változatlan. Ha a parity validation fail → rollback.

---

## Pre-existing Bug

A jelenlegi `mapEbAbout` TS mapper `asString()` hívást végez a repeater row objektumokon (`{text: "..."}`), ami `undefined`-ot ad, mivel `typeof {text: "..."} !== 'string'`. A dev frontend a `site.ts`-ből kapja az adatot közvetlenül, ezért működik. A WP REST endpoint-ról az adatok elvesznének. **Ez a migration ezt a bug-ot is fixálja**, mert a PHP builder a split után közvetlenül `string[]`-t ad ki.

---

## Érintett fájlok

| # | Fájl | Repó | Változás |
|---|------|------|----------|
| 1 | `infra/acf/sections/eb-about.php` | sp-exotica | repeater → textarea field def |
| 2 | `infra/acf/builders.php` | sp-exotica | `spektra_split_textarea()` helper + builder call |
| 3 | `infra/seed/mapping.ts` | sp-exotica | repeater extract → scalar join |

**NEM módosuló fájlok (explicit):**
- `src/data/wp-mapper.ts` — `mapEbAbout()` változatlan (string[]-t kap, helyesen feldolgozza)
- `src/data/normalize-site-data.ts` — `normalizeEbAbout()` változatlan
- `src/data/site.ts` — dev seed data változatlan (`string[]`)
- `src/sections/eb-about/eb-about.schema.ts` — `EbAboutData` interface változatlan
- `src/sections/eb-about/eb-about.component.tsx` — component változatlan

---

## Változás #1 — ACF field definition

**Fájl:** `sp-exotica/infra/acf/sections/eb-about.php`

**Before** (repeater + sub_fields blokk):
```php
[
    'key'        => 'field_eb_about_values',
    'label'      => 'Values',
    'name'       => 'eb_about_values',
    'type'       => 'repeater',
    'layout'     => 'table',
    'sub_fields' => [
        [
            'key'      => 'field_eb_about_values_text',
            'label'    => 'Value',
            'name'     => 'text',
            'type'     => 'text',
            'required' => 1,
        ],
    ],
],
```

**After** (egyetlen textarea):
```php
[
    'key'       => 'field_eb_about_values_text',
    'label'     => 'Values (soronként 1 érték)',
    'name'      => 'eb_about_values_text',
    'type'      => 'textarea',
    'rows'      => 5,
    'new_lines' => '',
],
```

**Megjegyzések:**
- A `key` a korábbi sub-field key-t használja (`field_eb_about_values_text`) — ez szándékos, elkerüli a korábbi repeater key-jel való ütközést
- `new_lines => ''` megakadályozza az ACF automatikus `<br>` vagy `<p>` wrapping-ját
- `rows => 5` — szerkesztési terület méret, nem tartalom limit

---

## Változás #2 — PHP builder

**Fájl:** `sp-exotica/infra/acf/builders.php`

### 2a — Helper function hozzáadása

A fájl elejére (a `require`-ök vagy function-ök elé) vagy a `spektra_build_eb_about` elé beileszteni:

```php
/**
 * Split multiline textarea into trimmed, non-empty string array.
 *
 * @param string $text Textarea content (newline-separated values).
 * @return string[]
 */
function spektra_split_textarea( string $text ): array {
	if ( $text === '' ) {
		return [];
	}
	return array_values( array_filter( array_map( 'trim', explode( "\n", $text ) ) ) );
}
```

**Megjegyzés:** A helper lokális marad az sp-exotica `builders.php`-ban. sp-infra-ba emelés csak akkor, ha legalább egy második migration is használja.

### 2b — Builder hívás módosítás

**Before:**
```php
'values'      => spektra_get_field( $p . 'values', $pid, [] ),
```

**After:**
```php
'values'      => spektra_split_textarea( spektra_get_field( $p . 'values_text', $pid, '' ) ),
```

**Megjegyzések:**
- `'values'` → `'values_text'`: az új ACF field name
- default `[]` → `''`: textarea default üres string, nem array
- `spektra_split_textarea()` végzi a split/trim/filter logikát
- A builder output típusa változatlan: `string[]`

---

## Változás #3 — Seed mapping

**Fájl:** `sp-exotica/infra/seed/mapping.ts`

**Before:**
```typescript
{
  acfKey: 'eb_about_values',
  kind: 'repeater',
  extract: (d) => {
    const values = d.values as string[]
    if (!Array.isArray(values)) return []
    return values.map((text) => ({ text: str(text) }))
  },
},
```

**After:**
```typescript
{
  acfKey: 'eb_about_values_text',
  kind: 'scalar',
  extract: (d) => {
    const values = d.values as string[]
    if (!Array.isArray(values)) return ''
    return values.join('\n')
  },
},
```

**Megjegyzések:**
- `kind: 'repeater'` → `kind: 'scalar'` — import-seed.php scalar path-et használja
- `map(text => ({text}))` → `join('\n')` — string[] → single newline-separated string
- empty fallback: `[]` → `''`
- `site.ts` NEM változik — továbbra is `values: string[]`, a mapping extract végzi a konverziót

---

## Data Flow After Migration

```
WP Admin:  textarea content "V1\nV2\nV3"
    ↓
PHP builder:  spektra_split_textarea() → ["V1", "V2", "V3"]
    ↓
REST endpoint:  {"values": ["V1", "V2", "V3"]}
    ↓
TS mapper:  asArray().map(asString).filter() → string[]  (NEM változik)
    ↓
SiteData:  values: ["V1", "V2", "V3"]  (VÁLTOZATLAN)
    ↓
Component:  values.map(v => <li>✦ {v}</li>)  (VÁLTOZATLAN)
```

---

## Seed Flow After Migration

```
site.ts:  values: ["V1", "V2", "V3"]
    ↓
mapping.ts extract:  values.join('\n') → "V1\nV2\nV3"
    ↓
seed.json:  "eb_about_values_text": "V1\nV2\nV3"
    ↓
import-seed.php:  detect_kind() → 'scalar'
    ↓
update_field('eb_about_values_text', "V1\nV2\nV3", $post_id)
```

---

## Parity Validation

**A migration után végrehajtandó (nem automatikus, manuális ellenőrzés):**

1. Before snapshot (migration előtt):
   ```bash
   wp eval-file import-seed.php
   curl -s .../wp-json/spektra/v1/site | jq '.sections[] | select(.type == "eb-about")' > before.json
   ```

2. Migration végrehajtás (ez a prompt)

3. Re-seed + After snapshot:
   ```bash
   wp eval-file import-seed.php
   curl -s .../wp-json/spektra/v1/site | jq '.sections[] | select(.type == "eb-about")' > after.json
   ```

4. Diff:
   ```bash
   diff before.json after.json
   ```

**Pass kritérium:** 0 diff az `eb-about` szekción belül. `values` array tartalma, sorrendje, és az összes egyéb mező azonos.

**Megjegyzés a pre-existing bug-ról:** Ha a before snapshot a WP endpoint-ról üres `values: []`-t ad (a mapper bug miatt), de a site.ts-ből 3 elemet vár: a migration UTÁN a WP endpoint helyesen `["V1","V2","V3"]`-t fog adni. Ebben az esetben a diff elvárt és helyes — a parity validation a **site.ts adatokkal** mint referenciával történik.

---

## Rollback

**Trigger:** Parity validation FAIL (kivéve a fent dokumentált pre-existing bug esetet) vagy bármi váratlan hiba.

**Lépések:**
1. `git revert <migration-commit>` — egyetlen commit revert
2. `wp eval-file import-seed.php` — repeater seed újrafuttatás
3. Endpoint ellenőrzés — before snapshot-tal egyezés

**Fontos:** Minden migration-változás egyetlen commitban. Ne keverd más munkával.

---

## Commit Convention

```
P12.4: migrate eb-about.values from ACF Repeater to textarea+split

- ACF field: repeater → textarea (eb_about_values_text)
- Builder: spektra_split_textarea() local helper + field name update
- Seed mapping: kind repeater → scalar, join('\n')
- SiteData output unchanged: string[]
- Fixes pre-existing mapper bug (asString on repeater row objects)
```

---

## Post-Migration Tasks

1. CHANGELOG frissítés (sp-docs)
2. `phase-12.3.md` P12.3a lezárás
3. `phase-12-strategy-overview.md` Phase Map frissítés: P12.4 DONE
4. Parity validation eredmény rögzítése
