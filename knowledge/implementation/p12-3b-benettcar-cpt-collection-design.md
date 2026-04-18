---
title: "P12.3b — BenettCar cpt_collection Design: bc-services.services"
status: design-complete
type: implementation
phase_introduced: "12.3b"
canonical: true
last_updated: 2026-04-18
tags: [cpt_collection, benettcar, bc-services, design, migration]
---

# P12.3b — BenettCar cpt_collection Design: bc-services.services

## Purpose

Az első `cpt_collection` source strategy design döntés dokumentálása. A P12.4-ben bizonyított `fixed_slots` pattern után ez a lépés a CPT-alapú collection loading tervezését végzi el — runtime implementáció NEM tartozik ide.

## Scope

- **Dokumentáció és design only** — runtime kód NEM módosul
- **Érintett repó:** sp-docs (primary), sp-benettcar és sp-infra kizárólag inspekció
- **Cél:** design döntések rögzítése, implementációs prompt előkészítés támogatása

---

## 1. Target

| | |
|---|---|
| **Selected target** | sp-benettcar `bc-services.services` |
| **Current source** | ACF Repeater field `bc_services_services` |
| **Future source** | CPT collection (`sp_bc_service`) |
| **Section type** | `bc-services` |
| **Client** | sp-benettcar |

### Scope kiegészítés

A `bc-services.services` az eredeti P12.3 Candidate B listában nem szerepelt — ott `bc-gallery.images`, `bc-brand.brands` és `bc-team.members` volt jelölve. Azonban a P12.2 Repeatable Content Source Strategy elfogadott classifikációja szerint `bc_services_services` is `cpt_collection` stratégiájú (ld. repeater field térkép #4). Az alacsonyabb implementációs kockázat (nincs image, nincs taxonomy, 3 scalar mező) miatt ez a legalkalmasabb első cpt_collection pilot target.

### Current ACF Repeater definition

```php
'key'        => 'field_bc_services_services',
'label'      => 'Services',
'name'       => 'bc_services_services',
'type'       => 'repeater',
'required'   => 1,
'min'        => 1,
'layout'     => 'block',
'sub_fields' => [
    ['name' => 'title',       'type' => 'text',     'required' => 1],
    ['name' => 'icon',        'type' => 'text',     'required' => 1],
    ['name' => 'description', 'type' => 'textarea', 'required' => 1, 'rows' => 3],
],
```

### Current SiteData output shape

```typescript
services: Array<{
  title: string
  icon: string
  description: string
}>
```

**Ez a shape NEM változik.**

---

## 2. CPT Naming Proposal

| | |
|---|---|
| **Proposed post type slug** | `sp_bc_service` |
| **Character count** | 14 (WP limit: 20) |
| **Prefix breakdown** | `sp` (Spektra-managed) + `bc` (client) + `service` (collection type) |

**Megjegyzés:**
- Ez az első CPT naming javaslat — nem globális naming convention
- Jövőbeli CPT slug-ok kötelesek a WordPress 20 karakter limitet betartani
- A `sp_` prefix egyértelműen jelzi, hogy Spektra-managed post type
- A `bc_` client prefix elkülöníti a kliensek post type-jait

### Slug limit ellenőrzés a többi target-re

| Field | Proposed slug | Chars | OK? |
|-------|--------------|-------|-----|
| `bc-services.services` | `sp_bc_service` | 14 | OK |
| `bc-brand.brands` | `sp_bc_brand` | 12 | OK |
| `bc-team.members` | `sp_bc_member` | 13 | OK |
| `bc-gallery.images` | `sp_bc_gallery` | 14 | OK |

---

## 3. Collection Loading Pattern

### Döntés

Kliens-lokális loader az első design-ban. Shared sp-infra abstrakció NEM készül.

**Proposed helper:** `spektra_bc_get_services()`

**Elhelyezés:** sp-benettcar kliens overlay (`infra/acf/builders.php` mellett vagy dedicated loaders fájl)

### Rationale

- Premature shared abstraction kerülése — egy kliens egy pattern-je nem elég a generalizáláshoz
- Először egy kliens-lokális implementáció bizonyítása
- sp-infra-ba emelés csak ismételt, validált használat után indokolt

### Pseudo-logic (NEM implementáció, design referencia)

```
spektra_bc_get_services():
  WP_Query( post_type => 'sp_bc_service', posts_per_page => -1, orderby => 'menu_order', order => 'ASC' )
  → array_map each post:
    title       = get_post_meta( $id, 'sp_service_title', true )
    icon        = get_post_meta( $id, 'sp_service_icon', true )
    description = get_post_meta( $id, 'sp_service_description', true )
  → return array of { title, icon, description }
```

### Builder integration (design referencia)

A `spektra_build_bc_services()` builder belső source-ja változik:

| | Before (repeater) | After (CPT) |
|---|---|---|
| **Data source** | `spektra_get_field($p . 'services', $pid)` | `spektra_bc_get_services()` |
| **Row iteration** | `array_map()` repeater sorokon | `array_map()` CPT post-okon |
| **Output shape** | `[{ title, icon, description }]` | `[{ title, icon, description }]` — **IDENTICAL** |

---

## 4. Builder Output Shape

**SiteData shape változatlan.**

Current builder output (és jövőben is):

```php
[
    'title'    => $title,
    'subtitle' => $subtitle,
    'services' => [
        [ 'title' => '...', 'icon' => '...', 'description' => '...' ],
        // ...
    ],
]
```

A frontend `bc-services` section props-ja NEM módosul. Csak a belső adat-source cserélődik.

---

## 5. Seed Strategy Impact

### Jelenlegi seed flow (repeater)

```
seed mapping: kind: 'repeater'
export: endpoint → repeater rows → seed.json array
import: update_field('bc_services_services', [...rows], $post_id)
```

### Jövőbeli seed flow (cpt_collection)

```
seed mapping: kind: 'cpt_collection'  (új kind)
export: WP_Query → CPT posts → seed.json array
import: wp_insert_post() + update_post_meta() per item
```

### Scope

- A seed support NEM tartozik ebbe a design lépésbe
- A seed pipeline cpt_collection támogatása **P12.5 — Seed Pipeline Source Strategy Support** scope-ja
- Az `import-seed.php` jelenleg nem tud `wp_insert_post()` hívást — P12.5 bővítés szükséges

---

## 6. Parity Validation Plan

### Before snapshot

```
GET /wp-json/spektra/v1/site
→ section type: 'bc-services'
→ data.services array rögzítése
```

### After migration

Azonos endpoint, azonos section output.

### Összehasonlítási kritériumok

| Szempont | Elvárt |
|----------|--------|
| Item count | azonos |
| Per-item `title` | byte-exact egyezés |
| Per-item `icon` | byte-exact egyezés |
| Per-item `description` | byte-exact egyezés |
| Item ordering | azonos sorrend |
| Section-level `title` | változatlan (ACF scalar, nem érintett) |
| Section-level `subtitle` | változatlan (ACF scalar, nem érintett) |
| SiteData shape | változatlan |

### Validation method

A P12.4-ben bizonyított WP-CLI parity validation pattern alkalmazandó:

```powershell
& $php $wp --path=$wpPath eval "
  do_action('rest_api_init');
  \$req = new WP_REST_Request('GET', '/spektra/v1/site');
  \$res = rest_do_request(\$req);
  \$sections = \$res->get_data()['sections'];
  foreach (\$sections as \$s) {
    if (\$s['type'] === 'bc-services') {
      echo json_encode(\$s['data'], JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
    }
  }
"
```

---

## 7. Rollback Plan

| Lépés | Művelet |
|-------|---------|
| 1 | Runtime commit revert (amikor implementáció történik) |
| 2 | Régi repeater adat megmarad a migration validálásáig |
| 3 | CPT posts törölhetők rollback esetén (`wp post delete` per `sp_bc_service` post) |
| 4 | ACF repeater source fallback-ként elérhető a migration fázisban |

**Megjegyzés:** A fallback lehetőség azt jelenti, hogy a builder ideiglenesen mindkét source-t próbálhatja — de ez implementációs döntés, nem design scope.

---

## 8. Deferred Targets

| Target | Deferral oka | Fő kockázat |
|--------|-------------|-------------|
| `bc-brand.brands` | Image/logo sub-field | image sideload + attachment ID kezelés CPT kontextusban |
| `bc-team.members` | Image + contact fields | image + phone/email meta, `spektra_normalize_media()` adaptation |
| `bc-gallery.images` | Image + category | image sideload + category → taxonomy lehetőség, `showCategories` flag |

**Sorrend ajánlás (implementációs komplexitás alapján):**
1. `bc-services.services` — 3 scalar ← **CURRENT**
2. `bc-brand.brands` — scalar + 1 image + bool
3. `bc-team.members` — scalar + 1 image + contact
4. `bc-gallery.images` — image + category/taxonomy (legkomplexebb)

---

## 9. Handover Impact Assessment

### Editorial UX változás

| | Repeater (jelenlegi) | CPT (jövőbeli) |
|---|---|---|
| **Szerkesztés** | Front page-en inline repeater sorok | Külön "Services" admin menüpont |
| **Új elem** | "Add Row" gomb a repeater-ben | "Add New" a Services CPT listában |
| **Sorrend** | Drag & drop a repeater-ben | `menu_order` vagy drag plugin |
| **Tanulási görbe** | Alacsony (repeater ismerős) | Közepes (új menüpont, új workflow) |

**Fontos:** A kliens számára ez workflow változás, nem adat veszteség. Az adat migrálódik a CPT post-okba.

---

## Summary

| Döntés | Eredmény |
|--------|----------|
| First cpt_collection target | `bc-services.services` |
| CPT post type slug | `sp_bc_service` |
| Loading pattern | Kliens-lokális `spektra_bc_get_services()` |
| SiteData shape | Változatlan |
| Seed support | Deferred → P12.5 |
| Parity validation | REST endpoint byte-exact match |
| Runtime implementáció | NEM ebben a lépésben — külön explicit prompt szükséges |
