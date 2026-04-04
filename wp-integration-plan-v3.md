# WordPress integráció — Terv v3

> **Megközelítés**: Platform-first framing, client-first implementation
> **Cél**: WP integráció a Benett Car klienshez, a Spektra platform újrafelhasználhatóságának megőrzésével
> **Státusz**: Tervezés
> **Dátum**: 2026-04-04

---

## Phase 0 — Platform-first framing

### P0: WP integration decision record

Rögzített döntések:

- A WordPress **opcionális integráció** — nem a Spektra core identity
- A **kanonikus adatstruktúra** továbbra is a `SiteData` (platform definiálja, nem a WP)
- A Benett Car az **első implementation target**, nem az egyetlen jövőbeli content source
- A WP **tartalomforrás** (content source), nem UI source — a frontend mindig a Spektra renderel, a WP csak JSON-t ad
- A `createWordPressAdapter` már létezik a platformban (`@spektra/data`), factory pattern-nel

---

### P0.1: Integration boundary definition

Mi hova tartozik:

**Platform** (sp-platform — section-agnosztikus, WP-agnosztikus):
- `SiteData`, `SiteMeta`, `Navigation`, `Page`, `Section`, `Media` típusok
- `SiteDataAdapter` interface (`load`, `revalidate`, `onError`)
- `createWordPressAdapter` factory (konfig-alapú, `mapResponse` DI)
- `createJsonAdapter` factory
- `validateSiteData` validation
- Normalization szabályok

**Client** (sp-clients/sp-benettcar — Benettcar-specifikus):
- 10 db `bc-*` section mapping + komponensek
- `wp-mapper.ts` — `mapResponse` implementáció (WP JSON → SiteData)
- Kliens-specifikus mezőnevek, content default-ok
- `.env` konfig (data source switch)
- `App.tsx` adapter wiring

**Infra** (sp-infra/sp-benettcar — WP + deploy környezet):
- WAMP / Docker setup
- WordPress telepítés + konfig
- `spektra-api` plugin (PHP)
- ACF field regisztráció
- Seed scriptek
- DB, plugin deployment

---

### P0.2: Canonical content contract

A `SiteData` mint elsődleges szerződés:

```
SiteData
├── site: SiteMeta
│   ├── name (kötelező)
│   ├── description?
│   ├── url?
│   └── locale?
├── navigation: Navigation
│   ├── primary: NavItem[] (kötelező, min 1)
│   │   └── { label, href, children?, external? }
│   └── footer?: NavItem[]
└── pages: Page[] (min 1)
    ├── slug (kötelező)
    ├── title?
    ├── meta?: PageMeta
    │   ├── title?
    │   ├── description?
    │   ├── canonical?
    │   └── ogImage?: Media
    └── sections: Section[] (min 1)
        ├── id (kötelező)
        ├── type (kötelező)
        ├── data: T (kötelező, section-specifikus)
        └── meta?: SectionMeta
```

**Media shape** (minden kép mező):
```
Media
├── src (kötelező) — teljes URL
├── alt (kötelező) — alt text
├── width? — pixel
├── height? — pixel
├── variants?: MediaVariant[]
│   ├── name
│   └── source: MediaSource
│       ├── url (kötelező)
│       ├── width?
│       ├── height?
│       └── format?
└── mimeType?
```

Fontos: `Media` gyökérszinten `src`, `MediaSource`-ban `url`.

**CallToAction shape**:
```
CallToAction
├── text (kötelező)
└── href?
```

**Section order rule**: a `pages[].sections` tömb sorrendje = megjelenítési sorrend.

---

### P0.3: Section payload policy (10 bc-*)

Szekciónként:

| type | required mezők | optional mezők | hiányzó mező viselkedés |
|---|---|---|---|
| `bc-hero` | title, backgroundImage | subtitle, description, primaryCTA, secondaryCTA | kép nélkül: fallback szín háttér |
| `bc-brand` | title, brands[] | description | üres brands: szekció elrejtés |
| `bc-gallery` | title, images[] | subtitle, showCategories | üres images: szekció elrejtés |
| `bc-services` | title, services[] | subtitle | üres services: szekció elrejtés |
| `bc-service` | title, services[] | subtitle, description, brands[], contact | — |
| `bc-about` | title, content[] | subtitle, image, imagePosition, stats[], cta, colorScheme | image nélkül: full-width text |
| `bc-team` | title, members[] | subtitle, description | üres members: szekció elrejtés |
| `bc-assistance` | title | subtitle, description, requestLabel, requestHref, serviceArea | — |
| `bc-contact` | title, contactInfo | subtitle, description, colorScheme | — |
| `bc-map` | query | title, height | height default: `450` |

**Általános szabály**: ha egy szekció `data`-jából a required mezők hiányoznak, a szekció **nem renderelődik** (skip, nem crash). Optional mező hiánya: degradált megjelenítés (kevesebb tartalom, de nincs törés).

---

### P0.4: Validation + normalization rules

A WP-ből jövő adat pipeline-ja:

```
raw WP JSON payload
  → mapResponse (kliens: wp-mapper.ts)
    → type cast + field mapping
    → normalization:
        - üres string → undefined
        - ACF image array → Media shape
        - hiányzó optional → kihagyás (nem null, nem "")
        - locale fallback → "hu-HU"
    → validation (kliens szintű: required mezők megvannak-e)
  → SiteData
  → validateSiteData (platform szintű: struktúra + típusok)
  → KÉSZ: érvényes SiteData
```

**Platform validation** (`validateSiteData`): struktúra-szintű (van-e `site.name`, `navigation.primary`, legalább 1 page, section-öknek van-e `id` + `type` + `data`). **Nem** validálja a `section.data` belső szerkezetét — az section-specifikus.

**Ha a WP integráció során kiderül, hogy a platform validation túl szigorú** (pl. üres `navigation.footer` hibát dob): a Platform Change Policy szerint módosítható.

---

### P0.5: Reusable extraction rule

Kategorizálás:

**Client-only** (soha nem kerül platformba):
- Konkrét Benettcar szövegek, képek
- 10 db `bc-*` section mezőelrendezés
- ACF field group definíciók
- Benettcar-specifikus `config.php`

**Potentially reusable** (ha második kliens jön, kiemelhető):
- `wp-mapper.ts` pattern (WP → SiteData konverzió logika)
- `class-response-builder.php` pattern (ACF → JSON assembly)
- Media normalization (ACF image → Media shape)
- Error handling stratégia (WP unavailable, partial content)

**Platform reusable** (már kész vagy bővíthető):
- `SiteDataAdapter` interface
- `createWordPressAdapter` factory
- `validateSiteData`
- Contract naming konvenció (`SiteData`, `Media`, `Navigation`)

---

## Platform Change Policy — WP Integration Phase

### Context

A Spektra platform még építés alatt áll. Platform módosítások megengedettek, ha javítják az architektúrát.

### Allowed Platform Changes

**Adapter contract módosítás**
Ha a WP integráció igényli: loading state, error state, retry logika, preview mód támogatás.
A `createWordPressAdapter` miatt ezek platformszintűvé válhatnak.

**Media contract módosítás**
WP image adatok: sizes, alt, mime, responsive variánsok.
Ha a jelenlegi `Media` shape nem elég — bővíthető, finomítható, normalizálható.

**Validation szabályok módosítása**
Ha szükséges: `validateSiteData` módosítható, validation lazítható, fallback policy bővíthető.

**Section payload rugalmasság**
Ha WP miatt szükséges: optional mezők, partial payload, dynamic mezők. Platform szinten módosítható.

### Not Allowed Platform Changes

**WordPress-specifikus logika a platformban**
Nem kerülhet a platformba: ACF logika, WP endpoint logika, WP image field logika, WP meta logika.
Ezek az adapter / mapper rétegben maradnak.

**Client-specifikus mezők a platformban**
Nem kerülhet: `bc-hero`, `bc-gallery`, `bc-contact`, stb. A platform section-agnosztikus marad.

### Decision Rule

- Workaround kell a kliensben → platform módosítás **megfontolandó**
- Mapping túl bonyolult → platform módosítás **megfontolandó**
- Validation túl szigorú → platform módosítás **megfontolandó**

### Goal

- WordPress integráció működjön
- Platform stabilabb legyen
- Új kliensekhez ne kelljen platform módosítás

---

## Phase 1 — Infra bootstrap

### P1.1: sp-infra mappaszerkezet

Létrehozandó: `D:\Projects\spektra\sp-infra\sp-benettcar\`

```
sp-infra\sp-benettcar\
├── wp\                     # WordPress runtime (WP fájlok)
├── plugin\                 # reusable WP integration core
│   └── spektra-api\
│       ├── spektra-api.php
│       └── includes\
│           ├── class-rest-controller.php
│           ├── class-response-builder.php
│           └── class-cors.php
├── config\                 # client-specific config
│   └── benettcar\
│       ├── config.php
│       └── acf-fields.php
├── docker\                 # Docker előkészítés
│   ├── docker-compose.yml
│   └── .env.example
├── seed\                   # Seed scriptek
│   ├── export-seed.ts
│   ├── package.json
│   └── seed.json           # generált, gitignored
└── docs\                   # Infra dokumentáció
```

**Szabályok**:
- `plugin/` = reusable integrációs mag (nincs kliens logika)
- `config/` = kliens-specifikus beállítások (benettcar)
- `wp/` = runtime only (WordPress fájlok)

---

### P1.2: Local environment ownership rule

Rögzített szabály:

- Elsődleges local runtime: **WAMP**
- Docker: **előkészített, de nem elsődleges** (prepared, not primary)
- Ha valaki a projektet átveszi: a WAMP setup a referencia
- A WAMP-on tesztelt állapot a "működő"
- Docker akkor lesz elsődleges, ha CI/CD vagy deploy igényli

---

### P1.3: docker-compose + .env.example

Előkészítés (nem indítás). Tartalom:

**docker-compose.yml**: WordPress 6.7 (PHP 8.3) + MariaDB 11.4 + WP-CLI service

```yaml
services:
  wordpress:
    image: wordpress:6.7-php8.3-apache
    ports: ["${WP_PORT:-8080}:80"]
    environment:
      WORDPRESS_DB_HOST: mariadb:3306
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      WORDPRESS_DEBUG: "true"
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_DEBUG_LOG', true);
        define('DISALLOW_FILE_EDIT', true);
    volumes:
      - wp-data:/var/www/html
      - ../plugin/spektra-api:/var/www/html/wp-content/plugins/spektra-api
      - wp-uploads:/var/www/html/wp-content/uploads
    depends_on:
      mariadb: { condition: service_healthy }

  mariadb:
    image: mariadb:11.4
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes: [db-data:/var/lib/mysql]
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 5s
      timeout: 5s
      retries: 5

  wpcli:
    image: wordpress:cli-php8.3
    volumes:
      - wp-data:/var/www/html
      - wp-uploads:/var/www/html/wp-content/uploads
      - ../seed:/seed
    depends_on: [wordpress]
    entrypoint: ["wp", "--allow-root"]
    profiles: ["cli"]

volumes:
  wp-data:
  db-data:
  wp-uploads:
```

**.env.example**:
```
MYSQL_DATABASE=spektra_benettcar
MYSQL_USER=spektra
MYSQL_PASSWORD=changeme
MYSQL_ROOT_PASSWORD=rootchangeme
WP_PORT=8080
```

---

### P1.4: WAMP vhost (benettcar.local)

Beállítandó:

1. **Apache vhost** (WAMP `httpd-vhosts.conf`):
   - `ServerName benettcar.local`
   - `DocumentRoot` → WP install mappa
2. **Windows hosts fájl**: `127.0.0.1 benettcar.local`
3. **Cél**: Vite (`localhost:5174`) és WP (`benettcar.local`) külön origin → valós CORS teszt

---

### P1.5: Filesystem + install path rule

Eldöntendő és rögzítendő:

| Opció | Leírás | Előny | Hátrány |
|---|---|---|---|
| A | WP közvetlenül `sp-infra/sp-benettcar/wp/`-ben | Minden egy helyen | WAMP-nak konfigolni kell ide |
| B | WP a WAMP `www/benettcar/`-ban, plugin symlink-elve `sp-infra`-ból | WAMP standard | Szétszórt fájlok |
| C | WP a WAMP `www/`-ban, az egész `sp-infra/sp-benettcar/wp/` symlink | Kompromisszum | Symlink platformfüggő |

A döntés a seed script, plugin fejlesztés és deploy szempontjából fontos.

---

## Phase 2 — Content contract to WP model

### P2.6: Content model spec (site.ts ↔ WP)

A jelenlegi `site.ts` alapján mezőtérkép. Szekciónként:

| Mező | field name (TS) | field type | req/opt | WP field name |
|---|---|---|---|---|
| bc-hero: cím | `title` | string | required | `bc_hero_title` |
| bc-hero: háttérkép | `backgroundImage` | Media | required | `bc_hero_background_image` |
| bc-hero: elsődleges CTA | `primaryCTA` | CallToAction | optional | `bc_hero_primary_cta` (group) |
| ... | ... | ... | ... | ... |

Teljes spec a 10 szekcióra, a `site.ts` source path-szel.

---

### P2.7: Section storage strategy (v1 döntés)

**Döntés (v1): Option B — Fixed Field Groups**

Követelmények:
- Implementációs cél: **single landing page (Home)**
- Architektúra: **multi-page bővíthető** később
- **Nem hardcode-oljuk** a single-page logikát a response builderben, mapperben vagy navigációban
- A response shape **marad**: `pages[] → sections[]`

Jövőbeli oldalak (később támogatandó):
- home
- about
- services
- contact

Ez azt jelenti:
- v1-ben egyetlen "Főoldal" page van
- a kód már most `pages[]`-re épül, nem "single page" shortcutokra

---

### P2.8: Media contract definition

Rögzített Media shape:

| Mező | Típus | Kötelező | Megjegyzés |
|---|---|---|---|
| `src` | string (URL) | igen | Teljes URL |
| `alt` | string | igen | Alt text |
| `width` | number | nem | Pixel |
| `height` | number | nem | Pixel |
| `variants` | MediaVariant[] | nem | Responsive méretek |
| `mimeType` | string | nem | Pl. `image/jpeg` |

**ACF image → Media konverzió** (PHP response builder-ben):

ACF "image (array return)" adja: `id`, `url`, `width`, `height`, `alt`, `sizes` (thumbnail, medium, large).
Ebből:
- `src` ← `url`
- `alt` ← `alt` (ha üres: fájlnév-ből generálni)
- `width` ← `width`
- `height` ← `height`
- `variants` ← `sizes` map (`name` + `MediaSource`)

Releváns blokkok: `bc-hero` (backgroundImage), `bc-brand` (brands[].logo), `bc-gallery` (images[]), `bc-about` (image), `bc-team` (members[].image), page meta (ogImage).

---

### P2.9: WP install + alap beállítás

WAMP-on:

1. WordPress letöltés (latest)
2. `wp-config.php` — DB credentials (WAMP MySQL)
3. Install wizard: site title, admin user, magyar nyelv
4. Beállítások:
   - Permalink: "Post name" (`/%postname%/`)
   - Időzóna: Europe/Budapest
   - Felesleges plugin-ok törlése (Hello Dolly, Akismet)
   - Default téma marad (headless, nem releváns)
5. Egy "Főoldal" page létrehozás ("Home" slug)
6. Settings → Reading: Static front page → "Főoldal"

---

### P2.10: ACF install + activation

1. ACF telepítés (free zip-ből vagy `wp plugin install`):
   - V1 döntés: **Fixed Field Groups** — **ACF free elég**
   - ACF Pro csak akkor kell, ha később flexible content-re váltunk
2. Aktiválás
3. Verifikáció: `acf_add_local_field_group` elérhető-e (PHP `function_exists` teszt)

---

### P2.11: ACF field registration (PHP)

A P2.6 spec alapján, `acf_add_local_field_group()` hívásokkal:

- Mind a 10 `bc-*` szekció-típus mezői
- Kódból regisztrálva (nem admin GUI) → verziózható, reprodukálható
- Fájl: `sp-infra/sp-benettcar/config/benettcar/acf-fields.php`
- Mező-típusok: text, textarea, wysiwyg, image, group (CTA: text + href), repeater (brands, services, members, images)
- Location rule: "Főoldal" page (vagy a P2.7 döntés szerinti CPT)

---

### P2.12: ACF registration verification

Ellenőrzés WP admin felületen:

1. "Főoldal" szerkesztő megnyitása → mezők megjelennek-e
2. Mező-típusok helyesek-e (kép feltöltés, repeater sorok hozzáadhatók)
3. PHP error log ellenőrzés (`wp-content/debug.log`) — nincs fatal
4. Load order: ACF init hook timing OK-e (a `spektra-api` plugin az ACF után tölt-e be)
5. Egy teszt adat beírása (pl. hero title) → mentés → újratöltés → megmarad

---

## Phase 3 — WP API delivery

### P3.13: spektra-api plugin skeleton

Minimum váz:

```
plugin/spektra-api/
├── spektra-api.php              # Plugin header, autoload, hook reg
└── includes/
    ├── class-rest-controller.php    # register_rest_route
    ├── class-response-builder.php   # SiteData JSON assembly
    └── class-cors.php               # CORS kezelés
```

Config külön mappában:
- `sp-infra/sp-benettcar/config/benettcar/config.php`
- `sp-infra/sp-benettcar/config/benettcar/acf-fields.php`

Szabályok:
- Plugin only — read-only API
- Nincs prezentációs logika
- Nincs kliens UI
- Generikus mag + kliens config réteg
- A plugin bootstrap a configot innen tölti: `../../config/benettcar/`
- A plugin mag nem tartalmaz kliens-hardcode-ot

---

### P3.14: REST route + CORS baseline

**Route**: `register_rest_route('spektra/v1', '/site', [...])`
- Method: GET
- Permission: public (nincs auth check, kivéve preview mód)
- Callback: `ResponseBuilder::build()`

**CORS** (`class-cors.php`):
- Allowed origins: `config/benettcar/config.php`-ból
  - Dev: `['http://localhost:5174', 'http://localhost:5175']`
  - Prod: `['https://benettcar.hu']`
- Headers: `Access-Control-Allow-Headers: Authorization, Content-Type`
- Methods: `GET, OPTIONS`
- OPTIONS preflight kezelés (204 válasz)
- `Access-Control-Allow-Credentials: true`
- `Access-Control-Max-Age: 86400`

**Auth stratégia**:
| Mód | URL | Auth |
|---|---|---|
| Publikus | `GET /wp-json/spektra/v1/site` | nincs |
| Preview | `GET /wp-json/spektra/v1/site?preview=true` | `Authorization: Bearer <token>` |

Publikus endpoint csak published tartalmat ad vissza.

---

### P3.15: Endpoint smoke test

Tesztelés (böngésző vagy curl):

```bash
curl -i http://benettcar.local/wp-json/spektra/v1/site
```

Ellenőrzés:
1. HTTP 200 (nem 404, nem 500)
2. `Content-Type: application/json`
3. Valid JSON válasz (akár minimális)
4. CORS header-ök (Vite origin-ről tesztelve a böngészőből)
5. Auth nélkül elérhető

Még nem kell teljes SiteData — elég egy minimum JSON, ami bizonyítja, hogy a route, a plugin és a CORS működik.

---

### P3.16: Response builder (ACF → SiteData)

A tényleges tartalom-összeállítás (`class-response-builder.php`):

1. **Site meta** összegyűjtés (WP `get_bloginfo` + options)
2. **Navigation** build (WP menük vagy ACF-ből)
3. **Page** kiolvasás ("Főoldal" page)
4. **Sections** assembly:
   - ACF field group-ok kiolvasása (`get_field`)
   - Szekciónként: mező értékek → section data shape
   - Media normalization: ACF image array → `Media` shape
5. **Page meta**: title, description, canonical, ogImage
6. **Végeredmény**: teljes SiteData-kompatibilis JSON

WP response shape minta:

```json
{
  "site": {
    "name": "Benett Car",
    "description": "Prémium autókereskedés",
    "url": "https://benettcar.hu",
    "locale": "hu-HU"
  },
  "navigation": {
    "primary": [
      { "label": "Főoldal", "href": "#hero" },
      { "label": "Galéria", "href": "#gallery" }
    ],
    "footer": [
      { "label": "Adatvédelem", "href": "/adatvedelem" }
    ]
  },
  "pages": [
    {
      "slug": "home",
      "title": "Főoldal",
      "meta": {
        "title": "Benett Car — Prémium autókereskedés",
        "description": "...",
        "canonical": "https://benettcar.hu",
        "ogImage": { "src": "...", "alt": "...", "width": 1200, "height": 630 }
      },
      "sections": [
        {
          "id": "hero",
          "type": "bc-hero",
          "data": {
            "title": "Benett Car",
            "subtitle": "...",
            "backgroundImage": { "src": "...", "alt": "...", "width": 1920, "height": 1080 },
            "primaryCTA": { "text": "Ajánlataink", "href": "#gallery" }
          }
        }
      ]
    }
  ]
}
```

---

## Phase 4 — Frontend integration

### P4.17: wp-mapper.ts + adapter boundary

`sp-clients/sp-benettcar/src/data/wp-mapper.ts`:

Nem csak mapping, hanem validated transform:
1. Raw WP payload → type assertion
2. Validation: required mezők megvannak-e
3. Normalization: üres string → undefined, hiányzó optional → kihagyás, locale fallback
4. Safe transform → `SiteData`

Utána a platform `createWordPressAdapter` a saját `validateSiteData`-val is ellenőrzi.

---

### P4.18: .env + App.tsx adapter switch

**Új fájlok** (sp-clients/sp-benettcar):

`.env.example`:
```
VITE_DATA_SOURCE=json
VITE_WP_API_BASE=http://benettcar.local
VITE_WP_AUTH_TOKEN=
```

`App.tsx` módosítás — env-alapú adapter switch:
- `VITE_DATA_SOURCE=json` → `createJsonAdapter({ data: siteData })` (jelenlegi viselkedés)
- `VITE_DATA_SOURCE=wordpress` → `createWordPressAdapter({ apiBase, mapResponse, auth? })`
- `site.ts` megmarad dev fallback-ként

**Fallback viselkedés**:
| Helyzet | Viselkedés |
|---|---|
| `json` mód | Statikus site.ts, WP nem kell |
| `wordpress` mód, WP elérhető | Éles WP adat |
| `wordpress` mód, WP nem elérhető | `onError` → hiba UI (nincs csendes JSON fallback) |

---

### P4.19: E2E dev validation (3 szekció)

Minimum vertical slice — 3 kritikus szekció:

1. **`bc-hero`** — kép (Media), szöveg, CTA → a legkomplexebb adat
2. **`bc-services`** — lista/repeater adat → array handling teszt
3. **`bc-contact`** — szöveges + kontakt info → egyszerűbb szöveg mezők

Tesztelési protocol:
1. WP adminban módosítás (pl. hero title)
2. Frontend frissítés (`VITE_DATA_SOURCE=wordpress`)
3. Változás megjelenik-e
4. CORS hibák nincsenek
5. Képek betöltődnek (Media shape helyes)
6. Nincs crash hiányzó optional mezőnél

---

### P4.20: Stabilization + fallback + doku

Csak a működő vertical slice (#19) után:

**Fallback UX**:
- Loading state (betöltés UI)
- WP unavailable state (hiba UI)
- Missing field fallback (degradált renderelés, nem crash)

**Maradék szekciók**:
- A maradék 7 `bc-*` szekció bekötése (bc-brand, bc-gallery, bc-about, bc-team, bc-assistance, bc-service, bc-map)

**Seed** (nem blocker, de ide tartozik):
- `export-seed.ts` — site.ts → seed.json
- WP-CLI import — seed.json → WP tartalom

**Dokumentáció**:
- Hogyan kell új klienst felvenni
- Content model módosítás workflow
- Plugin deployment guide
- Reusable extraction (mi emelhető ki a platformba)
