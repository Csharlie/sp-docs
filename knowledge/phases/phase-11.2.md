# Phase 11.2 — Section Builder Delegation

## Purpose

A P11.1 audit F1 (P1) finding-je azonosította: `sp-infra/acf/sections.php` 10 hardcoded `spektra_build_bc_*()` függvényt tartalmazott. Ez azt jelentette, hogy egy második kliens hozzáadásához shared infra forráskódot kellett volna módosítani — ez elfogadhatatlan, mert az sp-infra minden kliens számára közös infrastruktúra.

P11.2 célja: a section builder dispatch generikussá tétele az sp-infra-ban, a bc-* builder függvények áthelyezése a sp-benettcar kliens overlay-be, úgy hogy `class-response-builder.php` (az egyetlen consumer) signature-t nem módosítva marad.

---

## Commits

| Repo | Hash | Üzenet |
|---|---|---|
| sp-infra | `e85405b` | `refactor: P11.2 section builder delegation — generic dispatch` |
| sp-benettcar | `8f0286e` | `refactor: P11.2 move bc-* builders to client overlay` |

---

## Mi változott

### sp-infra/acf/sections.php — generic registry

**Eltávolítva:**
- 10 hardcoded `spektra_build_bc_*()` függvény definíció
- match-alapú dispatch (`match($type) { 'bc-hero' => spektra_build_bc_hero(...) ... }`)

**Hozzáadva:**

```php
// Generic builder registry
$GLOBALS['spektra_section_builders'] ??= [];

function spektra_register_section_builder( string $type, callable $callback ): void {
    $GLOBALS['spektra_section_builders'][ $type ] = $callback;
}

function spektra_get_section_data( string $type, int $post_id ): ?array {
    $builder = $GLOBALS['spektra_section_builders'][ $type ] ?? null;
    if ( ! $builder ) {
        return null;
    }
    $prefix = str_replace( '-', '_', $type ) . '_';
    return $builder( $post_id, $prefix );
}
```

- A prefix generálás (`str_replace('-', '_', $type) . '_'`) megmaradt, a dispatchba került
- `spektra_get_section_data()` signature **változatlan** — `class-response-builder.php` nem módosult

### sp-infra/plugin/spektra-api/spektra-api.php — overlay loading (+6 sor)

```php
// === Client section builders ===
$spektra_builders_path = dirname( $spektra_config_path ) . '/acf/builders.php';
if ( file_exists( $spektra_builders_path ) ) {
    require_once $spektra_builders_path;
}
```

- Ez a meglévő `field-groups.php` overlay loading precedenst követi (azonos `dirname($spektra_config_path)` mechanizmus)
- Ha a kliens overlay `acf/builders.php` nem létezik, a plugin csendben továbblép — nincs PHP error

### sp-clients/sp-benettcar/infra/acf/builders.php — ÚJ fájl

- 10 `spektra_build_bc_*()` függvény (szöveg szerinti áthelyezés `sp-infra/acf/sections.php`-ból)
- 10 `spektra_register_section_builder()` hívás a fájl végén — ez regisztrálja a buildereket a global registry-be

```php
// Példa regisztrációs blokk a fájl végén:
spektra_register_section_builder( 'bc-hero',       'spektra_build_bc_hero' );
spektra_register_section_builder( 'bc-brand',      'spektra_build_bc_brand' );
// ...
spektra_register_section_builder( 'bc-map',        'spektra_build_bc_map' );
```

---

## Nem módosult

### class-response-builder.php

A `\spektra_get_section_data($slug, $post_id)` hívás azonos — csak a mögöttes dispatch változott registry-alapúra. Ez a consumer-facing API stabilan maradt.

---

## Overlay Loading Mechanizmus

A kliens overlay betöltésének sorrendje a plugin bootstrap-ben:

```
spektra-api.php boot
  → require_once config.php           (kliens config: slug, origins, section lista)
  → require_once field-groups.php     (ACF field group definíciók)
  → require_once builders.php         (section builder regisztráció — P11.2 ÚJ)
  → register_rest_route(...)          (REST endpoint)
```

Az overlay fájlok path-ja: `dirname($spektra_config_path) . '/acf/<fajl>.php'`

Ez feltételezi, hogy:
1. A kliens overlay `infra/` könyvtárában `config.php` elsőként töltődik be (ez adja a `$spektra_config_path` értékét)
2. A `builders.php` ugyanabban a `acf/` alkönyvtárban van, mint a `field-groups.php`

---

## Registry Pattern Részletei

| Tulajdonság | Érték |
|---|---|
| **Registry tárolása** | `$GLOBALS['spektra_section_builders']` |
| **Lazy init** | `??= []` — null coalescing assignment, biztonságos többszörös include esetén |
| **Regisztráció** | `spektra_register_section_builder(string $type, callable $callback)` |
| **Lekérdezés** | `spektra_get_section_data(string $type, int $post_id): ?array` |
| **Prefix generálás** | `str_replace('-', '_', $type) . '_'` — pl. `bc-hero` → `bc_hero_` |
| **Builder nem találva** | `return null` — a response builder kihagyja a section-t |

Második kliens belépési pont: az új kliens overlay-ének `infra/acf/builders.php` fájlja regisztrálja a saját section buildereit — az sp-infra nem módosul.

---

## Validáció

| Check | Eredmény |
|---|---|
| `pnpm test` (sp-benettcar) | 50/50 PASS |
| `pnpm build` (sp-benettcar) | PASS |
| PHP lint — sections.php | PASS |
| PHP lint — spektra-api.php | PASS |
| PHP lint — builders.php (sp-benettcar) | PASS |
| Live WP endpoint (`wp-json/spektra/v1/site`) | 10 bc-* section, mind `DATA` státusz — PASS |

---

## Boundary Matrix — P11.2 utáni állapot

| Concern | Infra | Client overlay | Notes |
|---|---|---|---|
| Section builder registry | `acf/sections.php` — generic | — | Registry + dispatch az infra-ban |
| Section builder implementáció | — | `infra/acf/builders.php` | Kliens-owned, kliens-specifikus logika |
| Section builder regisztráció | — | `infra/acf/builders.php` | `spektra_register_section_builder()` hívások |
| Overlay betöltés | `spektra-api.php` | — | `dirname($spektra_config_path)` mechanizmus |
| Consumer API | `acf/sections.php` — `spektra_get_section_data()` | — | Signature változatlan |

---

## F1 Finding Lezárása

A P11.1 F1 (P1) finding lezárt:

> **Volt:** `sp-infra/acf/sections.php` 10 hardcoded `spektra_build_bc_*()` függvényt tartalmazott. Második kliens = shared infra módosítás.
>
> **Most:** `sp-infra/acf/sections.php` csak generic dispatch-et tartalmaz. A bc-* builderek `sp-clients/sp-benettcar/infra/acf/builders.php`-ban élnek. Második kliens a saját `infra/acf/builders.php` overlay-en keresztül regisztrál — sp-infra nem módosul.

A P11.1 boundary matrix `Section building` sora frissítve: `P11.2: resolved — builder delegation pattern`.

---

## Érintett fájlok

| Fájl | Változás |
|---|---|
| `sp-infra/acf/sections.php` | Generic registry + dispatch — bc-* builderek kivéve |
| `sp-infra/plugin/spektra-api/spektra-api.php` | +6 sor: overlay `builders.php` betöltése |
| `sp-clients/sp-benettcar/infra/acf/builders.php` | ÚJ: 10 bc-* builder + 10 regisztrációs hívás |
| `sp-infra/plugin/spektra-api/includes/class-response-builder.php` | Nem módosult |

---

## Következő fázisok

| Fázis | Cél | P11.1 finding |
|---|---|---|
| P11.3 | Overlay template — második kliens számára másolható sablon | F2 (P1) |
| P11.4 | Infra script paraméterezés (`seed-pipeline.ps1`, `verify-endpoint.php`) | F3, F4 (P2) |
| P11.5 | New client onboarding guide | F2 (P1) |

---

## Done State

- [x] sp-infra/acf/sections.php: nincs bc-* hardcode
- [x] sp-infra generic registry + dispatch
- [x] sp-benettcar overlay builders.php létezik és regisztrál
- [x] spektra-api.php betölti az overlay builders.php-t
- [x] class-response-builder.php nem módosult
- [x] 10 bc-* section live endpoint-on DATA státusz
- [x] 50/50 teszt PASS
- [x] F1 (P1) finding lezárva
