# Spektra Plugin Architecture Guide

> Célközönség: junior fejlesztők, senior fejlesztők, külső kontribútorok.
> Utolsó frissítés: 2026-04-05
> **Implementációs státusz: Phase 8 closed** — HTTP layer + Response Builder + Contract Hardening kész (P8-R4).

---

## Tartalomjegyzék

1. [Mi a Spektra plugin szerepe](#mi-a-spektra-plugin-szerepe)
2. [High Level Architecture](#high-level-architecture)
3. [Fájlstruktúra magyarázat](#fájlstruktúra-magyarázat)
4. [Request Flow](#request-flow)
5. [Boundary szabályok](#boundary-szabályok)
6. [CORS működés](#cors-működés)
7. [Config rendszer](#config-rendszer)
8. [Design Principles](#design-principles)
9. [Hogyan bővítsd a plugint](#hogyan-bővítsd-a-plugint)
10. [Future Phases](#future-phases)
11. [Összefoglaló](#összefoglaló)

---

## Mi a Spektra plugin szerepe

A Spektra egy **headless** rendszer. Három rétege van:

| Réteg | Mi ez | Hol él |
|---|---|---|
| **Content source** | WordPress + ACF | `benettcar.local` (Apache) |
| **Adapter** | Spektra API plugin | `sp-infra/plugin/spektra-api/` |
| **Renderer** | React + Vite frontend | `localhost:5174` |

A plugin az **adapter**: kiszedi a tartalmat a WordPress-ből, és JSON formátumban
kiadja a frontend számára.

### Mit csinál a plugin

- REST endpoint-ot regisztrál (`/spektra/v1/site`)
- ACF mezőkből kiolvas tartalmat
- `SiteData`-kompatibilis JSON-t épít
- CORS headereket kezel (cross-origin hozzáférés)
- Kliens config-ot tölt be overlay-ből

### Mit NEM csinál a plugin

- **Nem renderel UI-t** — nincs HTML, nincs template, nincs shortcode
- **Nem tud a frontendről** — nem tudja, hogy React, Vue vagy Angular rendereli
- **Nem ír adatbázisba** — csak olvas
- **Nem tartalmaz kliens-specifikus logikát** — azt az overlay adja

Ez a szétválasztás szándékos. A plugin egy **generikus adapter**, ami bármely Spektra
kliensnél működik — a kliens-specifikus rész a config overlay-ben él.

---

## High Level Architecture

```
┌──────────────────────────────────────────────────┐
│  Frontend (Vite / React)                         │
│  localhost:5174                                   │
│  fetch("/wp-json/spektra/v1/site")               │
└───────────────────────┬──────────────────────────┘
                        │ HTTP GET
                        ▼
┌──────────────────────────────────────────────────┐
│  class-cors.php                                  │
│  Origin whitelist → CORS headerek                │
│  Preflight (OPTIONS) → 204                       │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│  class-rest-controller.php                       │
│  Route: GET /spektra/v1/site                     │
│  Param validáció → Response Builder delegálás    │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│  class-response-builder.php                      │
│  ACF mezők → SiteData JSON összeállítás          │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│  WordPress / ACF                                 │
│  get_field(), get_option(), WP Query             │
└──────────────────────────────────────────────────┘
```

Minden réteg **egyetlen felelősséggel** rendelkezik:

- **CORS** → HTTP security layer (ki férhet hozzá?)
- **Controller** → HTTP routing layer (hogyan jön a kérés, hogyan megy a válasz?)
- **Builder** → Data assembly layer (milyen adat megy ki?)
- **WordPress / ACF** → Data source (honnan jön az adat?)

---

## Fájlstruktúra magyarázat

```
wp-content/plugins/
├── spektra-api/              ← sp-infra/plugin/spektra-api/ (Junction)
│   ├── spektra-api.php                 ← belépési pont
│   └── includes/
│       ├── class-rest-controller.php   ← route + request kezelés
│       ├── class-cors.php              ← CORS headerek
│       └── class-response-builder.php  ← JSON összeállítás
│
└── spektra-config/           ← sp-benettcar/infra/ (Junction)
    ├── config.php                      ← kliens beállítások
    └── acf/
        └── field-groups.php            ← ACF mező definíciók
```

### spektra-api.php — belépési pont

**Szerep:**
- Plugin bootstrap — WordPress felismeri mint plugint (Plugin Name header)
- Konstansok definiálása: `SPEKTRA_API_VERSION`, `SPEKTRA_API_PATH`, `SPEKTRA_API_URL`
- Include-ok betöltése — a három osztály `require_once`-szal
- Kliens config betöltése — `config.php` → `SPEKTRA_CLIENT_CONFIG` konstans
- ACF field group betöltése — overlay `acf/field-groups.php`
- Hook regisztráció — `rest_api_init` action-re Controller + CORS

**Miért fontos:**
- Ez a WordPress **egyetlen belépési pontja** a pluginba
- A WordPress lifecycle-ban a plugin fájl az `plugins_loaded` előtt fut
- A `rest_api_init` hook biztosítja, hogy a route regisztráció a megfelelő pillanatban történik
- A config betöltés **strategy pattern**: ENV var → symlink fallback

```php
$spektra_config_path = getenv( 'SPEKTRA_CLIENT_CONFIG' )
    ?: WP_PLUGIN_DIR . '/spektra-config/config.php';
```

**Mit NEM csinál:**
- Nem tartalmaz üzleti logikát
- Nem olvas ACF mezőket
- Nem küld HTTP választ

---

### class-rest-controller.php — route és request kezelés

**Szerep:**
- Route regisztráció: `GET /wp-json/spektra/v1/site`
- Request param validáció (preview param — csak `'true'` elfogadott)
- Response wrapper — `WP_REST_Response` objektum létrehozása
- HTTP header kezelés — `X-Spektra-Version`, `Cache-Control: no-cache` (preview)

**A route regisztráció:**

```php
register_rest_route( 'spektra/v1', '/site', [
    'methods'             => 'GET',
    'callback'            => [ self::class, 'handle_request' ],
    'permission_callback' => '__return_true',
    'args'                => [
        'preview' => [
            'type'              => 'string',
            'required'          => false,
            'validate_callback' => [ self::class, 'validate_preview_param' ],
            'sanitize_callback' => 'sanitize_text_field',
        ],
    ],
]);
```

**Validáció:**
- A `preview` param **strict**: csak `'true'` string elfogadott
- Minden más érték (`1`, `yes`, `on`, üres string) → `WP_Error` 400
- Ha nincs preview param → normál mód

**Response headerek:**
- `X-Spektra-Version` — mindig, a plugin verziót jelzi
- `Cache-Control: no-cache` — csak preview módban (a publikus endpoint-ra nem vonatkozik cache policy)

**Mit NEM csinál:**
- Nem olvas ACF mezőket — delegálja a Response Builder-nek
- Nem épít JSON-t — a Builder adja vissza az adatot
- Nem kezel CORS-t — az a CORS class felelőssége

**Miért fontos a szétválasztás:** ha a route logika és az adatgyűjtés egy helyen lenne,
minden módosítás (új param, új header, új section) egyetlen fájlban halmozódna.

---

### class-cors.php — CORS kezelés

**Szerep:**
- Cross-origin hozzáférés kezelése
- Origin whitelist ellenőrzés (`SPEKTRA_CLIENT_CONFIG['allowed_origins']`)
- Preflight OPTIONS → 204 No Content válasz
- WordPress default CORS felülírása disallowed origin-nél

**Filter priority: 100** (WP default CORS: priority 10)

Ez szándékos: a WordPress default `rest_send_cors_headers()` priority 10-en fut és
`Access-Control-Allow-Origin: <bármilyen origin>`-t küld. A Spektra CORS filter priority
100-on **felülírja** ezt: engedélyezett origin → saját header, nem engedélyezett origin
→ `header_remove()`.

**Scope:** Csak `/spektra/` route-okra vonatkozik. Más WP REST endpoint-ok megtartják
a WordPress default CORS viselkedését.

**Mit NEM csinál:**
- Nem kezel autentikációt — az a `permission_callback` dolga
- Nem módosítja a response body-t
- Nem WP REST endpoint-okat általánosan — csak a Spektra namespace-t

**Miért külön fájl:**
- HTTP security layer — más felelősség, mint a routing
- A CORS logika független a business logikától
- Külön tesztelhető, külön módosítható

---

### class-response-builder.php — JSON összeállítás

**Szerep:**
- WordPress adatok kiolvasása (ACF mezők, site beállítások)
- `SiteData`-kompatibilis JSON tömb összeállítása
- Preview mód támogatása (draft tartalom)

**Jelenlegi állapot: ✅ Kész (Phase 7)**

A `build()` metódus valódi `SiteData` JSON-t állít elő: site meta, navigation, section assembly
ACF mezőkből, média normalizálás. A config forrás a `SPEKTRA_CLIENT_CONFIG` konstans (P8-R4).

**Miért instance method (nem static):**
- A builder állapotot tarthat (cached fields, preview flag)
- Tesztelhetőbb — mock-olható, injectable

**A builder:
- ACF mezők kiolvasása field group-onként
- Site meta összeállítás (név, locale, URL)
- Navigation struktúra építés
- Section-ök assembly-je a kliens config `sections` listája alapján
- Média normalizálás (ACF image → `{ src, alt, width, height }`)

---

## Request Flow

Lépésről lépésre, mi történik egy kérés során:

### 1. Frontend kérés

```
React app mount → adapter.load() → fetch("http://benettcar.local/wp-json/spektra/v1/site")
```

A böngésző észreveszi, hogy cross-origin kérés (`:5174` → `:80`).

### 2. Apache → WordPress

Az Apache a `benettcar.local` vhost-on fogadja a kérést. A WordPress `index.php`
betöltődik, a plugin-ok aktiválódnak, a `spektra-api.php` lefut.

### 3. Hookok lefutnak

A `rest_api_init` action-ben két dolog történik:
- `Rest_Controller::register_routes()` → a `/spektra/v1/site` route létrejön
- `CORS::register_hooks()` → a `rest_pre_serve_request` filter regisztrálva

### 4. Route matching

A WordPress REST router megtalálja a `/spektra/v1/site` route-ot.
A `preview` param validálva (ha van) → `validate_preview_param()`.

### 5. Controller handler

`handle_request()` lefut:
- Preview param kiolvasás
- `Response_Builder::build()` hívás
- `WP_REST_Response` létrehozás
- Headerek beállítása (`X-Spektra-Version`, esetleg `Cache-Control`)

### 6. CORS filter

A `rest_pre_serve_request` filter lefut (priority 100):
- Route ellenőrzés: `/spektra/`? → igen, folytatás
- Origin ellenőrzés: whitelist-ben van? → header beállítás vagy eltávolítás
- Preflight? → 204

### 7. Válasz

```json
HTTP/1.1 200 OK
Content-Type: application/json
X-Spektra-Version: 0.1.0
Access-Control-Allow-Origin: http://localhost:5174
Vary: Origin

{ "site": [], "navigation": [], "pages": [] }
```

### Vizuálisan

```
Browser (localhost:5174)
    │
    ├── [preflight?] OPTIONS → CORS → 204
    │
    └── GET /wp-json/spektra/v1/site
            │
            ▼
        Apache → WordPress → plugins_loaded
            │
            ▼
        rest_api_init
            ├── Rest_Controller::register_routes()
            └── CORS::register_hooks()
            │
            ▼
        Route match → validate_preview_param()
            │
            ▼
        handle_request()
            ├── Response_Builder::build($is_preview)
            ├── X-Spektra-Version header
            └── [preview?] Cache-Control: no-cache
            │
            ▼
        rest_pre_serve_request (priority 100)
            ├── /spektra/ route? → yes
            ├── origin in whitelist? → Allow-Origin + Vary
            └── [OPTIONS?] → 204 + preflight headers
            │
            ▼
        JSON response → Browser → React render
```

---

## Boundary szabályok

A plugin architektúra szigorú boundary-ket tart. Ez a legfontosabb szekció a
kontribútoroknak.

### A plugin IGEN:

| Felelősség | Miért |
|---|---|
| Adatot olvas WordPress-ből | Ez az adapter szerepe |
| JSON-t épít SiteData formátumban | A szerződés PHP oldala |
| HTTP headereket kezel | CORS, cache, verzió |
| REST route-ot regisztrál | WordPress REST framework-ön belül |
| Kliens config-ot betölt | Overlay pattern |
| Input-ot validál és sanitize-ol | Biztonsági minimum |

### A plugin NEM:

| Tilalom | Miért |
|---|---|
| Nem renderel HTML-t / UI-t | Headless — a frontend renderel |
| Nem tud a frontend technológiáról | Nem tudja, hogy React, Vue, vagy más |
| Nem tartalmaz platform logikát | Platform = sp-platform repo |
| Nem ír adatbázisba | Read-only adapter |
| Nem tartalmaz kliens-specifikus kódot | Az overlay-ben él |
| Nem küld email-t, nem loggol fájlba | Nem az ő felelőssége |
| Nem módosít más plugin viselkedését | Kivéve: WP default CORS override saját route-okon |

### Fizikai határok

```
sp-infra/                    ← Plugin forráskód (reusable)
    plugin/spektra-api/      ← EZ a plugin
sp-benettcar/                ← Kliens config (kliens-specifikus)
    infra/                   ← EZ az overlay
sp-platform/                 ← Frontend app (React + Vite)
    apps/                    ← EZ a renderer
sp-platform/                 ← Platform packages (TypeScript + React)
    packages/types/src/site.ts ← EZ a szerződés
```

Egy fájl **soha nem importál** másik repo-ból közvetlenül. A kapocsok:
- Plugin ↔ Overlay: `config.php` betöltés futásidőben
- Plugin ↔ Frontend: HTTP (REST API JSON válasz)
- Plugin ↔ Engine: a `SiteData` szerződés (TypeScript ↔ PHP — emberi szinkronban tartva, forrás: `sp-platform/packages/types/`)

---

## CORS működés

### Miért kell

A frontend és a backend különböző origin-en fut:

| Oldal | Origin |
|---|---|
| Frontend | `http://localhost:5174` |
| Backend | `http://benettcar.local` |

A böngésző Same-Origin Policy-je alapértelmezetten **blokkolja** a cross-origin fetch-et.
A CORS headerek engedélyezik.

### Hogyan működik

1. A böngésző elküldi a kérést az `Origin` headerrel
2. A `class-cors.php` ellenőrzi: az origin benne van az `allowed_origins` listában?
3. **Igen** → `Access-Control-Allow-Origin: <origin>` + `Vary: Origin`
4. **Nem** → `header_remove()` (WP default CORS eltávolítása) + `Vary: Origin`
5. **Preflight (OPTIONS)** → 204 + Allow-Methods, Allow-Headers, Max-Age

### Allowed origins

A whitelist a kliens config-ból jön:

```php
// sp-benettcar/infra/config.php
'allowed_origins' => ['http://localhost:5174'],
```

Production-ben ez a kliens domain-je lesz (pl. `https://benettcar.hu`).

### WordPress CORS override

A WordPress default `rest_send_cors_headers()` priority 10-en **minden origin-re**
küld `Allow-Origin` headert. A Spektra CORS filter priority 100-on fut és:
- Saját route-okon (`/spektra/`) felülírja
- Más WP route-okon nem nyúl hozzá

Részletek: [cors-guide.md](cors-guide.md)

---

## Config rendszer

### Client overlay

A plugin **nem tartalmaz kliens-specifikus kódot**. Minden kliens-specifikus beállítás
egy overlay-ben él:

```
spektra-config/           ← Junction → sp-benettcar/infra/
├── config.php            ← kliens beállítások
└── acf/
    └── field-groups.php  ← ACF mező definíciók
```

### Config betöltés

Két stratégia:

| Stratégia | Megvalósítás | Mikor |
|---|---|---|
| ENV var | `SPEKTRA_CLIENT_CONFIG=/path/to/config.php` | Production, Docker |
| Symlink fallback | `plugins/spektra-config/` Junction | Lokális dev (WAMP) |

A plugin nem tudja és nem is kell tudnia, melyik stratégia aktív.

### Config tartalma

```php
return [
    'client_slug'     => 'benettcar',
    'client_name'     => 'Benett Car',
    'allowed_origins' => ['http://localhost:5174'],
    'site_defaults'   => ['lang' => 'hu', 'title' => 'Benett Car'],
    'sections'        => [
        'bc-hero', 'bc-brand', 'bc-gallery', 'bc-services', 'bc-service',
        'bc-about', 'bc-team', 'bc-assistance', 'bc-contact', 'bc-map',
    ],
];
```

| Kulcs | Ki használja | Mire |
|---|---|---|
| `client_slug` | Response Builder | Site meta azonosító |
| `client_name` | Response Builder | Site meta megjelenítő név |
| `allowed_origins` | CORS class | Origin whitelist |
| `site_defaults` | Response Builder | Alapértelmezett locale, title |
| `sections` | Response Builder | Mely ACF section-öket kell összegyűjteni |

A config a `SPEKTRA_CLIENT_CONFIG` PHP konstansba kerül — bármely osztály közvetlenül
eléri: `SPEKTRA_CLIENT_CONFIG['allowed_origins']`.

---

## Design Principles

A Spektra plugin 5 tervezési elvre épül:

### 1. Platform Agnostic

A plugin **nem tud a frontendről**. Nem tudja, hogy React, Vue, Svelte, vagy mobil app
olvassa az API-t. JSON-t ad ki, és kész.

**Teszt:** ha kicserélnéd a React app-ot egy Angular app-ra, a plugin kódja **egyetlen
sorban sem változna**.

### 2. Client Configurable

A plugin viselkedése a **config overlay-ből** származik, nem a forráskódból.
Új kliens = új overlay, nem új plugin fork.

**Teszt:** ha holnap egy „ClientX" kliensnek kellene API, a plugin kódja **változatlan**
marad — csak a `spektra-config/` Junction mutat máshova.

### 3. Minimal Logic

A plugin minimális logikát tartalmaz. Nem transzformál, nem aggregál, nem cache-el,
nem optimalizál. Kiolvas, összerak, kiad.

**Miért:** a komplexitás a frontendé (presentation logic) és a WordPress-é (content
management). A plugin csak a híd.

### 4. Stable Contract

A plugin kimenete (`SiteData` JSON) a **szerződés**. Ha a kimenet formátuma változik,
a frontend elromlik. Ezért:
- A schema fix (site, navigation, pages, sections)
- Új mezők hozzáadhatók, de meglévők nem törölhetők
- Verziókezelés: `spektra/v1` → ha breaking change kell, jön a `v2`

### 5. Separation of Concerns

Minden fájlnak **egyetlen felelőssége** van:

| Fájl | Egyetlen felelősség |
|---|---|
| `spektra-api.php` | Bootstrap, config, hookok |
| `class-rest-controller.php` | HTTP routing, params, headers |
| `class-cors.php` | Cross-origin security |
| `class-response-builder.php` | Adat → JSON |

Ha egy módosítás több fájlt érint, az valószínűleg **boundary violation**.

---

## Hogyan bővítsd a plugint

### Új endpoint hozzáadása

Ha a jövőben kell egy második endpoint (pl. `/spektra/v1/health`):

1. `class-rest-controller.php` → `register_routes()`-ba új `register_rest_route()` hívás
2. Új handler metódus a Controller-ben (vagy új class, ha komplex)
3. A CORS class automatikusan érvényes rá (a `/spektra/` prefix match miatt)

```php
register_rest_route( self::API_NAMESPACE, '/health', [
    'methods'             => 'GET',
    'callback'            => [ self::class, 'handle_health' ],
    'permission_callback' => '__return_true',
]);
```

### Response Builder bővítés

A `build()` metódus belsejébe kerülnek az adatgyűjtő lépések:

```php
public function build( bool $is_preview = false ): array {
    return [
        'site'       => $this->build_site_meta(),
        'navigation' => $this->build_navigation(),
        'pages'      => $this->build_pages( $is_preview ),
    ];
}
```

Minden `build_*` metódus **egyetlen felelősséggel** rendelkezik.

### Config bővítés

Ha a plugin új config kulcsot igényel:

1. Adj hozzá a `sp-benettcar/infra/config.php`-hoz (overlay)
2. Olvasd ki: `SPEKTRA_CLIENT_CONFIG['new_key'] ?? $default`
3. **Ne módosítsd a plugin kódot** — a config tömb nyitott, a plugin nullable-ként olvassa

### Új kliens beüzemelés

```
1. mkdir sp-clients/sp-newclient/infra/
2. Készíts config.php-t (slug, name, origins, sections)
3. Készíts acf/field-groups.php-t (kliens-specifikus mezők)
4. Junction: plugins/spektra-config → sp-newclient/infra/
5. Kész — /spektra/v1/site az új kliens adatait adja
```

A plugin forráskód: **változatlan**.

---

## Future Phases

> A jelenlegi állapot: **Phase 8 closed (P8-R4)**. Phase 6–8 lezárva.

### Phase 6 — ACF Validation ✅

- Reusable ACF helper függvények (`spektra_get_field`, `spektra_get_section_data`)
- Media helper (`spektra_normalize_media`)
- ACF field group regisztráció verifikálás
- Admin felületi ellenőrzés
- Teszt adat bevitel + persistence teszt

### Phase 7 — Response Builder ✅

- `Response_Builder::build()` valódi implementáció
- Site meta + Navigation assembly
- Section assembly — ACF mezők → `SiteData.pages[].sections[]`
- Media normalization — ACF image → `{ src, alt, width, height }`
- Teljes SiteData endpoint teszt

### Phase 8 — Contract Hardening ✅ (P8-R4)

- CTA contract: `href` required (nem opcionális)
- Render-safety: normalize-site-data.ts — 50/50 teszt
- Config boundary: Response Builder `SPEKTRA_CLIENT_CONFIG` konstansból olvas
- Fallback DOM: `<div>` wrapper `<span>` helyett
- Docs alignment: content-model, API guide, plugin architecture guide

---

## Összefoglaló

| Kérdés | Válasz |
|---|---|
| Hány fájl a plugin? | 4 fájl (`spektra-api.php` + 3 class) |
| Hány endpoint? | 1 (`GET /spektra/v1/site`) |
| Hol van a kliens config? | Overlay-ben (`spektra-config/config.php`) |
| Ki kezeli a CORS-t? | `class-cors.php`, priority 100, origin whitelist |
| Ki építi a JSON-t? | `class-response-builder.php` (✅ Kész — Phase 7) |
| Milyen headerek mennek? | `X-Spektra-Version` (mindig), `Cache-Control: no-cache` (preview), CORS headerek |
| Hogyan bővíthető? | Új endpoint = új route, új kliens = overlay csere, új adat = Builder bővítés |
| Mi a tervezési elv? | Platform agnostic, client configurable, minimal logic, stable contract, separation of concerns |
