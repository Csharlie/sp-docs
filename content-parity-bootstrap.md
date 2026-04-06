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
| Navigation (label, href) | Out of scope | Config-driven (config.php), nem ACF — lásd §6.6 |
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
| P8.5.4 | Minimal Implementation | Implementation | export-seed.ts, mapping.ts, import-seed.php |
| P8.5.5 | Verification | Implementation | verify-parity.ts, PASS eredmény |
| P8.5.6 | Documentation | Docs | Postmortem, workflow standard, v4 plan update |

## 6. Tooling Design (P8.5.3)

### 6.1. seed.json shape

A seed.json ACF-aware struktúrájú fájl. Két szekció:
`site_options` (WP settings), `sections` (ACF field data — szekciónként csoportosítva,
mapping kind-dal annotálva: scalar / repeater / group / image).

Navigation **nem** szerepel a seed-ben — config-driven (lásd §6.6).

```json
{
  "post_id": "front_page",
  "site_options": {
    "blogname": "Benett Car",
    "blogdescription": "Autószerviz, felvásárlás és útmenti segítség Cegléden."
  },
  "sections": {
    "bc-hero": {
      "bc_hero_title": { "value": "Precíz munka. Valódi odafigyelés.", "kind": "scalar" },
      "bc_hero_subtitle": { "value": "Volkswagen Konszern...", "kind": "scalar" },
      "bc_hero_description": { "value": "Minden munkát az adott autóra szabunk...", "kind": "scalar" },
      "bc_hero_primary_cta_text": { "value": "Szerviz egyeztetés", "kind": "scalar" },
      "bc_hero_primary_cta_href": { "value": "#car-service", "kind": "scalar" },
      "bc_hero_background_image": { "value": "https://images.unsplash.com/...", "kind": "image" },
      "bc_hero_background_image_alt": { "value": "Autószerviz műhely", "kind": "scalar" }
    },
    "bc-brand": {
      "bc_brand_title": { "value": "Volkswagen Konszern...", "kind": "scalar" },
      "bc_brand_brands": {
        "kind": "repeater",
        "value": [
          { "name": "Volkswagen", "logo": "https://...", "alt": "Volkswagen logó", "invert": true }
        ]
      }
    }
  }
}
```

**Szabályok:**
- `post_id: "front_page"` → import-seed.php feloldja `get_option('page_on_front')`-ból
- `kind` annotáció: az importer ebből tudja, melyik ACF write path-ot használja
- Image mezők: URL string + külön `_alt` mező (P8.5.4 bővítés)
- Repeater mezők: teljes sor-tömb — `update_field()` kezeli a row indexelést

### 6.2. ACF field mapping (10 szekció)

A tábla a site.ts data key → ACF field name leképezést mutatja.
A PHP builder-ek (`sp-infra/acf/sections.php`) pontosan ezeket a field name-eket olvassák.

#### bc-hero

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_hero_title` | text | ✅ |
| `subtitle` | `bc_hero_subtitle` | text | — |
| `description` | `bc_hero_description` | textarea | ✅ |
| `primaryCTA.text` | `bc_hero_primary_cta_text` | text | — |
| `primaryCTA.href` | `bc_hero_primary_cta_href` | text | — |
| `secondaryCTA.text` | `bc_hero_secondary_cta_text` | text | — |
| `secondaryCTA.href` | `bc_hero_secondary_cta_href` | text | — |
| `backgroundImage.src` | `bc_hero_background_image` | image (URL) | — |
| `backgroundImage.alt` | `bc_hero_background_image_alt` | text | — |

#### bc-brand

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_brand_title` | text | — |
| `description` | `bc_brand_description` | textarea | — |
| `brands[N].name` | `bc_brand_brands_N_name` | text | — |
| `brands[N].logo` | `bc_brand_brands_N_logo` | image (URL) | — |
| `brands[N].alt` | `bc_brand_brands_N_alt` | text | — |
| `brands[N].invert` | `bc_brand_brands_N_invert` | true_false | — |

Repeater: `bc_brand_brands` (required — hiánya → section skip)

#### bc-gallery

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_gallery_title` | text | ✅ |
| `subtitle` | `bc_gallery_subtitle` | text | — |
| `showCategories` | `bc_gallery_show_categories` | true_false | — |
| `images[N].src` | `bc_gallery_images_N_src` | url | — |
| `images[N].alt` | `bc_gallery_images_N_alt` | text | — |
| `images[N].category` | `bc_gallery_images_N_category` | text | — |
| `images[N].caption` | `bc_gallery_images_N_caption` | text | — |

Repeater: `bc_gallery_images` (required — hiánya → section skip)

#### bc-services

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_services_title` | text | ✅ |
| `subtitle` | `bc_services_subtitle` | text | — |
| `services[N].title` | `bc_services_services_N_title` | text | — |
| `services[N].icon` | `bc_services_services_N_icon` | text | — |
| `services[N].description` | `bc_services_services_N_description` | textarea | — |

Repeater: `bc_services_services` (required — hiánya → section skip)

#### bc-service

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_service_title` | text | ✅ |
| `subtitle` | `bc_service_subtitle` | text | — |
| `description` | `bc_service_description` | textarea | ✅ |
| `services[N].label` | `bc_service_services_N_label` | text | — |
| `brands[N]` | `bc_service_brands_N_name` | text | — |
| `contact.title` | `bc_service_contact_title` | text | — |
| `contact.description` | `bc_service_contact_description` | textarea | — |
| `contact.phone` | `bc_service_contact_phone` | text | — |
| `contact.messageCta.text` | `bc_service_contact_message_cta_text` | text | — |
| `contact.messageCta.href` | `bc_service_contact_message_cta_href` | text | — |
| `contact.bookingNote` | `bc_service_contact_booking_note` | textarea | — |
| `contact.hours` | `bc_service_contact_hours` | text | — |
| `contact.weekendHours` | `bc_service_contact_weekend_hours` | text | — |

Repeater: `bc_service_services` (required), `bc_service_brands` (required — site.ts `string[]` → ACF `[{name}]`)
Group: `bc_service_contact` (optional — hiánya nem skipeli a szekciót)

#### bc-about

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_about_title` | text | ✅ |
| `subtitle` | `bc_about_subtitle` | text | — |
| `content[N]` | `bc_about_content_N_paragraph` | textarea | — |
| `image.src` | `bc_about_image` | image (URL) | — |
| `image.alt` | `bc_about_image_alt` | text | — |
| `imagePosition` | `bc_about_image_position` | select | — |
| `colorScheme` | `bc_about_color_scheme` | select | — |
| `stats[N].value` | `bc_about_stats_N_value` | text | — |
| `stats[N].label` | `bc_about_stats_N_label` | text | — |
| `cta.text` | `bc_about_cta_text` | text | — |
| `cta.href` | `bc_about_cta_href` | text | — |

Repeater: `bc_about_content` (required — site.ts `string[]` → ACF `[{paragraph}]`)

#### bc-team

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_team_title` | text | ✅ |
| `subtitle` | `bc_team_subtitle` | text | — |
| `description` | `bc_team_description` | textarea | — |
| `members[N].name` | `bc_team_members_N_name` | text | — |
| `members[N].role` | `bc_team_members_N_role` | text | — |
| `members[N].image` | `bc_team_members_N_image` | image (URL) | — |
| `members[N].image.alt` | `bc_team_members_N_image_alt` | text | — |
| `members[N].phone` | `bc_team_members_N_phone` | text | — |
| `members[N].email` | `bc_team_members_N_email` | text | — |

Repeater: `bc_team_members` (required — hiánya → section skip)

#### bc-assistance

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_assistance_title` | text | ✅ |
| `subtitle` | `bc_assistance_subtitle` | text | — |
| `description` | `bc_assistance_description` | textarea | — |
| `requestLabel` | `bc_assistance_request_label` | text | — |
| `requestHref` | `bc_assistance_request_href` | text | — |
| `serviceArea` | `bc_assistance_service_area` | text | — |

#### bc-contact

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_contact_title` | text | ✅ |
| `subtitle` | `bc_contact_subtitle` | text | — |
| `description` | `bc_contact_description` | textarea | — |
| `colorScheme` | `bc_contact_color_scheme` | select | — |
| `contactInfo.phone` | `bc_contact_info_phone` | text | — |
| `contactInfo.email` | `bc_contact_info_email` | text | — |
| `contactInfo.address` | `bc_contact_info_address` | text | — |

Group: `bc_contact_info` (optional)

#### bc-map

| site.ts key | ACF field | Típus | Required |
|---|---|---|---|
| `title` | `bc_map_title` | text | — |
| `query` | `bc_map_query` | text | ✅ |
| `height` | `bc_map_height` | number | — |

### 6.3. export-seed CLI interface

```
npx tsx export-seed.ts [options]

Options:
  --output, -o    Output path (default: ../../sp-infra/seed/seed.json)
  --dry-run       Print seed.json to stdout, don't write file
  --verbose       Print field-by-field mapping log
```

A script közvetlenül importálja a `site.ts`-t (`import { siteData } from '../../src/data/site'`),
és a `mapping.ts`-ben definiált konverziós logikát használja.

**Nem kell `--input` vagy `--mapping` flag** — a kliens repóban van, tehát a
site.ts és mapping.ts relative importtal elérhető.

### 6.4. import-seed.php interface

PHP-alapú importer — WordPress kontextusban fut, ACF API-t (`update_field()`) használ.

```bash
wp eval-file import-seed.php [seed.json path] [options]

Arguments:
  seed.json       Path to seed file (default: ./seed.json)

Options:
  --dry-run       Print operations without executing
  --verbose       Print each field update
```

**ACF write stratégia kind szerint:**

| Kind | Mechanika | Példa |
|---|---|---|
| `scalar` | `update_field('bc_hero_title', $value, $post_id)` | text, textarea, select, url, number |
| `repeater` | `update_field('bc_brand_brands', $rows_array, $post_id)` | ACF kezeli a row count-ot + sub-field indexelést |
| `group` | `update_field('bc_service_contact', $assoc_array, $post_id)` | ACF kezeli a sub-field-eket |
| `image` | `update_field('bc_hero_background_image', $url, $post_id)` | URL stringként — return format = url |

**Site options:** `update_option('blogname', $value)` (nem ACF, WP core).`

### 6.5. Image handling — media exception

Az export-seed.ts az image URL-eket **stringként** írja a seed.json-ba,
a hozzájük tartozó alt textet külön `_alt` mezőben.
Az import-seed.php `update_field()`-del írja be mindkettőt.

Ez működik, mert:
- A 3 érintett ACF field group (bc-hero, bc-about, bc-team) bővítve lett
  külön `_alt` mezővel (P8.5.4)
- A PHP builder-eket is frissítjük, hogy olvassák az alt mezőt
- A `spektra_normalize_media()` URL stringet is elfogad, de most már
  az alt text is elérhető a külön mezőből
- A parity check (verify-parity.ts) media exception: URL eltérhet,
  alt text és struktúra egyeznie kell

**Nem-goal:** WP media library attachment import. Ez P11+ scope.

### 6.6. Navigation — scope döntés

A navigáció **nem** a parity scope része. Indoklás:
- A Response Builder a `config.php`-ból épít navot, nem a WP DB-ből
- Csak primary-t támogatja; a site.ts-ben footer is van
- A navigáció config-driven marad (Phase 11.5: native WP menu integráció)
- A seed.json nem tartalmaz navigation blokkot
- A parity check a navigation-t kihagyja

## 7. Phase 9 blokkolás

Phase 9 (E2E section bekötés) **NEM blokkolt** — párhuzamosan futhat.
De az adapter runtime switch production cutover TILOS a P8.5.5 PASS nélkül.

## 8. Új kliens onboarding checklist (Phase 11+ előkészítés)

1. `site.ts` → kliens tartalom megírása
2. `infra/seed/mapping.ts` → ACF field mapping (kind annotációval)
3. `infra/seed/export-seed.ts` → seed generálás
4. `import-seed.php` → WP import (ACF API)
5. `verify-parity.ts` → PASS
6. Adapter switch engedélyezése
