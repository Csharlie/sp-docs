# Spektra WordPress Integration Baseline v4

> **Megközelítés**: Platform-first framing, client-first implementation
> **Cél**: Multi-client, multi-source, multi-page platform — WordPress mint opcionális tartalomforrás
> **Státusz**: Architecture baseline
> **Verzió**: v4 (v3 alapján bővített)
> **Dátum**: 2026-04-04

---

## 1. Core Principles

A Spektra platform architektúrájának alapelvei:

| # | Elv | Leírás |
|---|---|---|
| 1 | **Platform-agnostic** | A platform nem függ WordPress-től, nem függ semmilyen CMS-től |
| 2 | **WordPress = opcionális content source** | A WP egy lehetséges tartalomforrás, nem az egyetlen |
| 3 | **Frontend = Spektra rendered** | A frontend mindig a Spektra renderel — a WP soha |
| 4 | **WP = JSON data source only** | A WordPress kizárólag JSON API-n keresztül szolgáltat adatot |
| 5 | **Adapter boundary required** | Minden külső forrás kizárólag adapter-en keresztül éri el a platformot |
| 6 | **SiteData = canonical contract** | A `SiteData` a platform egyetlen igazságforrása — minden adapter ezt produkálja |
| 7 | **Clean boundary based** | Platform / Client / Infra határ megbonthatatlan |
| 8 | **Multi-client first** | Az architektúra több kliens egyidejű működésére van tervezve |
| 9 | **Expandable** | Új forráspár, page, szekció, kliens hozzáadható anélkül, hogy a platform változna |
| 10 | **Reusable** | Minden generikus elem újrafelhasználható — kliens-specifikus elem soha nem kerül a platformba |

---

## 2. Architecture Goals

### Elsődleges célok

- **Multi-client**: több kliens párhuzamosan, közös platformon, saját tartalomforrásokkal
- **Multi-source**: egy kliens több tartalomforrást használhat (JSON, WordPress, headless CMS, API)
- **Multi-page**: `pages[]` → `sections[]` architektúra — nem single-page hardcode
- **Platform-expandable**: új adapter, új section type, új validátor hozzáadása platform-módosítás nélkül
- **WordPress optional**: WP integráció teljesen opcionális — a platform nélküle is működik

### Architektúra-minőségi célok

- Platform-agnosztikus (nincs CMS import a platformban)
- Bővíthető (új kliens = új repo, nem platform fork)
- Újrafelhasználható (plugin, ACF helper, seed pipeline közös)
- Tiszta határokon alapuló (boundary = repo határ + DI)

---

## 3. v3 Integration Model (Preserved)

A v4 a v3 minden döntését **megtartja és terjeszti ki**. Az alábbi v3 elemek változatlanul érvényesek:

### 3.1. Adapter pattern

A platform `SiteDataAdapter` interface-t definiál:

```ts
interface SiteDataAdapter {
  load(): Promise<SiteData>
  init?(): Promise<void>
  revalidate?(): Promise<SiteData>
  onError?(error: unknown): void
}
```

Két beépített factory:
- `createJsonAdapter(config)` — statikus/URL JSON forrás
- `createWordPressAdapter(config)` — WP REST API forrás, `mapResponse` dependency injection

### 3.2. Validation pipeline (v3 → preserved)

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

### 3.3. Canonical SiteData contract (v3 → preserved)

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

### 3.4. Media shape (v3 → preserved)

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

**Fontos**: `Media` gyökérszinten `src`, `MediaSource`-ban `url`. Ez tervezési döntés, nem hiba.

### 3.5. Section storage (v3 → preserved)

**Döntés**: Fixed Field Groups (ACF free).
- Implementációs cél v1: **single landing page (Home)**
- Architektúra: **multi-page bővíthető** később
- Response shape: `pages[] → sections[]`
- Kódból regisztrált ACF field group-ok (nem admin GUI)

### 3.6. Section payload policy — 10 bc-* (v3 → preserved)

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

**Általános szabály**: required mezők hiánya → szekció **nem renderelődik** (skip, nem crash). Optional hiánya → degradált megjelenítés.

### 3.7. Seed strategy (v3 → preserved)

```
site.ts → export-seed.ts → seed.json → WP-CLI import → WordPress
```

### 3.8. Auth strategy (v3 → preserved)

| Mód | URL | Auth |
|---|---|---|
| Publikus | `GET /wp-json/spektra/v1/site` | nincs |
| Preview | `GET /wp-json/spektra/v1/site?preview=true` | `Authorization: Bearer <token>` |

### 3.9. Platform Change Policy (v3 → preserved)

**Megengedett platform módosítások** (WP integráció során):
- Adapter contract bővítés (loading state, retry, preview mód)
- Media contract finomítás (responsive variánsok, normalization)
- Validation szabály lazítás (ha a WP integráció indokolja)
- Section payload rugalmasság (optional mezők, partial payload)

**Nem megengedett**:
- WordPress-specifikus logika a platformban (ACF, WP endpoint, WP image field)
- Client-specifikus mezők a platformban (bc-hero, bc-gallery, stb.)

**Döntési szabály**: ha workaround kell a kliensben, **megfontolandó** a platform módosítás.

---

## 4. Repository Architecture

### 4.1. Repository típusok

| Repo | Típus | Jelleg | Példa |
|---|---|---|---|
| **sp-platform** | Shared | Platform core — típusok, adapterek, validáció, témák | `@spektra/types`, `@spektra/data` |
| **sp-infra** | Shared | Reusable infra — plugin, ACF helpers, Docker, seed, scripts | `plugin/spektra-api/`, `acf/` |
| **sp-benettcar** | Client | Kliens app — frontend, kliens mapping, kliens infra overlay | `src/`, `data/`, `infra/` |
| **sp-docs** | Shared | Dokumentáció | Architecture docs, integration plans |

### 4.2. Repository határ szabály

- **Shared repo** = soha nem tartalmaz kliens-specifikus kódot
- **Client repo** = soha nem tartalmaz platform-generikus kódot
- **sp-infra** = reusable eszközök, amiket minden kliens használ — de kliens config **SOHA**
- **Client infra overlay** = kliens-specifikus infra config, ami a kliens repo `infra/` mappájában él

### 4.3. Mappaszerkezet

```
D:\Projects\spektra\
├── sp-platform/             # SHARED — platform core
│   └── packages/
│       ├── types/           # SiteData, Media, Navigation, stb.
│       ├── data/            # createWordPressAdapter, createJsonAdapter, validateSiteData
│       └── themes/          # Tailwind preset-ek
│
├── sp-infra/                # SHARED — reusable infra
│   ├── plugin/              # spektra-api WP plugin (reusable core)
│   ├── acf/                 # Reusable ACF helpers
│   ├── docker/              # Docker base config
│   ├── seed/                # Seed pipeline tools
│   ├── scripts/             # Tooling (symlink, bootstrap, env loader)
│   ├── apps/                # Runnable infra presets (WAMP config, stb.)
│   └── docs/                # Infra documentation
│
├── sp-clients/
│   └── sp-benettcar/        # CLIENT — Benettcar kliens
│       ├── src/             # React frontend
│       ├── data/            # site.ts (dev fallback)
│       ├── docs/            # Client-specific docs
│       └── infra/           # CLIENT INFRA OVERLAY
│           ├── config.php   # Kliens-specifikus WP config
│           ├── acf/         # Kliens ACF field groups
│           ├── env/         # .env.example, environment files
│           └── docs/        # Infra overlay documentation
│
├── sp-docs/                 # SHARED — documentation
└── .local/                  # GITIGNORED — local runtimes
    └── wp-runtimes/
        └── benettcar/       # Assembled WP runtime
```

---

## 5. Platform / Client / Infra Boundary

### 5.1. Boundary definíció

| Réteg | Tartalom | Repo | WP-ismeret |
|---|---|---|---|
| **Platform** | Típusok, adapterek, validáció, témák | sp-platform | ❌ WP-agnosztikus |
| **Client** | Frontend, section mapping, kliens config | sp-benettcar | 🟡 Adapter-en keresztül |
| **Infra** | WP runtime, plugin, ACF, Docker, seed | sp-infra + client/infra/ | ✅ WP-specifikus |

### 5.2. Dependency flow

```
sp-platform  ← nem függ semmitől (core)
sp-infra     ← nem függ sp-platform-tól (PHP, independent)
sp-benettcar ← függ sp-platform-tól (@spektra/types, @spektra/data)
              ← NEM függ sp-infra-tól (runtime-ban a WP biztosítja az adatot)
```

### 5.3. Platform réteg tartalma (sp-platform)

Section-agnosztikus, WP-agnosztikus:

| Package | Export | Leírás |
|---|---|---|
| `@spektra/types` | `SiteData`, `SiteMeta` | Kanonikus adatstruktúra |
| | `Navigation`, `NavItem` | Navigáció |
| | `Page`, `PageMeta` | Oldalak |
| | `Section`, `SectionMeta`, `SectionType` | Szekciók (generikus) |
| | `Media`, `MediaVariant`, `MediaSource` | Média |
| | `CallToAction` | CTA descriptor |
| | `SiteDataAdapter` | Adapter interface |
| | `ThemeConfig`, `ThemeColors`, `ThemeFonts` | Téma |
| `@spektra/data` | `createWordPressAdapter` | WP adapter factory |
| | `createJsonAdapter` | JSON adapter factory |
| | `validateSiteData` | Struktúra-szintű validáció |

### 5.4. Client réteg tartalma (sp-benettcar)

Benettcar-specifikus:

| Elem | Leírás |
|---|---|
| 10 db `bc-*` section komponens | React komponensek |
| `wp-mapper.ts` | WP JSON → SiteData konverzió (`mapResponse`) |
| `site.ts` | Statikus dev data (JSON adapter fallback) |
| `.env` | Data source switch (`VITE_DATA_SOURCE=json\|wordpress`) |
| `App.tsx` | Adapter wiring |
| `infra/` | Client infra overlay (config, ACF fields, env) |

### 5.5. Infra réteg tartalma

Két helyen él:

**sp-infra** (shared, reusable):
- `plugin/spektra-api/` — generikus WP REST plugin
- `acf/` — reusable ACF helper függvények
- `docker/` — Docker base config
- `seed/` — Seed pipeline eszközök
- `scripts/` — Tooling
- `apps/` — Runnable presets

**sp-benettcar/infra/** (client overlay):
- `config.php` — kliens CORS origins, site name, stb.
- `acf/` — 10 db `bc-*` field group definíció
- `env/` — environment fájlok

---

## 6. sp-infra Architecture

### 6.1. Mappaszerkezet

```
sp-infra/
├── plugin/
│   └── spektra-api/
│       ├── spektra-api.php              # Plugin header, autoload, hooks
│       └── includes/
│           ├── class-rest-controller.php # register_rest_route
│           ├── class-response-builder.php# SiteData JSON assembly
│           └── class-cors.php           # CORS + preflight kezelés
│
├── acf/
│   ├── helpers.php                      # Reusable ACF field helpers
│   ├── media-helper.php                 # ACF image → Media shape konverzió
│   └── README.md                        # Helper documentation
│
├── docker/
│   ├── docker-compose.yml               # WP + MariaDB + WP-CLI
│   └── .env.example                     # DB credentials template
│
├── seed/
│   ├── export-seed.ts                   # site.ts → seed.json export
│   ├── import-seed.sh                   # WP-CLI import script
│   └── package.json                     # Seed tool dependencies
│
├── scripts/
│   ├── bootstrap.ps1                    # Full local setup script
│   ├── link-plugin.ps1                  # Symlink plugin → runtime
│   ├── link-overlay.ps1                 # Symlink client overlay → runtime
│   └── setup-env.ps1                    # Environment loader
│
├── apps/
│   └── wamp/
│       ├── vhost.conf                   # Apache vhost template
│       └── README.md                    # WAMP setup guide
│
└── docs/
    ├── plugin-architecture.md
    ├── acf-conventions.md
    ├── runtime-assembly.md
    └── new-client-guide.md
```

### 6.2. Szabályok

| Szabály | Leírás |
|---|---|
| **Nincs kliens logika** | sp-infra-ban NEM szerepel `benettcar`, `bc-*`, kliens-specifikus config |
| **Plugin = reusable core** | A `spektra-api` plugin generikus — a kliens config-ot kívülről kapja |
| **ACF helpers = utility** | Az `acf/` mappa helper függvényeket tartalmaz, nem field group definíciókat |
| **Config loading** | A plugin a kliens config-ot path-ból tölti (env var vagy symlink) |
| **scripts/ = PowerShell** | Windows-first fejlesztés (WAMP), Bash alternatíva opcionális |

### 6.3. Plugin config loading mechanizmus

A `spektra-api` plugin nem hardcode-olja a kliens config elérési útját. Két stratégia:

**Stratégia A — Symlink**:
```
runtime/wp-content/plugins/spektra-api/      → sp-infra/plugin/spektra-api/
runtime/wp-content/plugins/spektra-api-config/ → sp-benettcar/infra/
```

**Stratégia B — ENV var**:
```php
// spektra-api.php
$config_path = getenv('SPEKTRA_CLIENT_CONFIG') ?: __DIR__ . '/../spektra-api-config/';
require_once $config_path . 'config.php';
```

A végleges stratégia a P5.4 lépésben dől el.

---

## 7. Client Infra Overlay Model

### 7.1. Koncepció

A kliens-specifikus infra konfiguráció a **kliens repo-ban** él, nem az sp-infra-ban. Ez biztosítja:

- Kliens repo = self-contained (minden ami a kliens működéséhez kell, egy helyen)
- sp-infra = tisztán reusable (nincs kliens szennyeződés)
- Több kliens ≠ sp-infra conflict

### 7.2. Overlay mappa struktúra

```
sp-benettcar/
└── infra/
    ├── config.php              # Kliens WP config
    │   - plugin name override
    │   - CORS allowed origins
    │   - site default values
    │
    ├── acf/
    │   ├── field-groups.php    # Minden bc-* field group regisztráció
    │   └── sections/           # Szekciónkénti ACF definíciók (opcionális split)
    │       ├── bc-hero.php
    │       ├── bc-brand.php
    │       └── ...
    │
    ├── env/
    │   └── .env.example        # WP environment template
    │
    └── docs/
        └── content-model.md    # Kliens content model dokumentáció
```

### 7.3. Overlay szabályok

| Szabály | Leírás |
|---|---|
| **Overlay = kliens repo része** | Nem külön repo, hanem `sp-benettcar/infra/` |
| **Overlay = nem futtatható** | Nem indítható önmagában — a runtime-ba kell linkelni |
| **Overlay = verziózott** | Git-ben követve, a kliens repo-val együtt commitolva |
| **Overlay fájlok = PHP** | A WP plugin PHP-t vár — az overlay is PHP |
| **Overlay ≠ runtime** | Az overlay config fájlokat tartalmaz, nem WP fájlokat |

---

## 8. WordPress Runtime Model

### 8.1. Koncepció

A WordPress runtime **nem source-of-truth**. A runtime egy assembled, gitignored könyvtár, ami a következőkből áll össze:

```
WordPress core files (letöltés)
  + sp-infra/plugin/spektra-api/ (symlink)
  + sp-benettcar/infra/ (symlink)
  + ACF plugin (WP plugin install)
  = Működő WP runtime
```

### 8.2. Runtime helye

```
D:\Projects\spektra\.local\wp-runtimes\benettcar\
├── wp-admin/
├── wp-content/
│   ├── plugins/
│   │   ├── spektra-api/     → SYMLINK: sp-infra/plugin/spektra-api/
│   │   ├── spektra-config/  → SYMLINK: sp-benettcar/infra/
│   │   └── advanced-custom-fields/
│   ├── uploads/
│   └── themes/
├── wp-includes/
└── wp-config.php
```

### 8.3. Runtime szabályok

| Szabály | Leírás |
|---|---|
| **Gitignored** | `.local/` teljes mappa gitignored — nincs version control |
| **Assembled** | Nem kézzel hozod létre — script állítja össze |
| **Reproducible** | `bootstrap.ps1` script-tel bármikor újra felépíthető |
| **WAMP primary** | Az Apache DocumentRoot erre a mappára mutat |
| **Symlink-based** | Plugin és config symlink-ek — fejlesztés közben az eredeti fájlt szerkeszted |
| **Nem klónozható** | A runtime nem commitolható, nem deployolható — mindig assembled |

### 8.4. WAMP konfiguráció

```apache
<VirtualHost *:80>
    ServerName benettcar.local
    DocumentRoot "D:/Projects/spektra/.local/wp-runtimes/benettcar"
    <Directory "D:/Projects/spektra/.local/wp-runtimes/benettcar">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Windows hosts fájl: `127.0.0.1 benettcar.local`

### 8.5. Docker alternatíva (előkészített, nem elsődleges)

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

---

## 9. ACF Architecture

### 9.1. Kétszintű ACF modell

| Szint | Hely | Tartalom |
|---|---|---|
| **Reusable helpers** | `sp-infra/acf/` | ACF utility függvények (image → Media, repeater helpers) |
| **Client field groups** | `sp-benettcar/infra/acf/` | 10 db `bc-*` section field group definíció |

### 9.2. Reusable ACF helpers (sp-infra/acf/)

```php
// helpers.php — generikus ACF utility-k

function spektra_acf_image_to_media(array $image): array {
    // ACF image (array return) → Spektra Media shape
    return [
        'src'    => $image['url'],
        'alt'    => $image['alt'] ?: pathinfo($image['filename'], PATHINFO_FILENAME),
        'width'  => $image['width'],
        'height' => $image['height'],
        'variants' => spektra_acf_sizes_to_variants($image['sizes'] ?? []),
        'mimeType' => $image['mime_type'] ?? null,
    ];
}

function spektra_acf_sizes_to_variants(array $sizes): array {
    // ACF sizes → MediaVariant[]
    // 'thumbnail', 'medium', 'large' stb.
}

function spektra_acf_group_to_cta($group): ?array {
    // ACF group field → CallToAction shape (vagy null ha üres)
}
```

### 9.3. Client field groups (sp-benettcar/infra/acf/)

```php
// field-groups.php — Benettcar-specifikus ACF regisztráció

add_action('acf/init', function () {
    // bc-hero
    acf_add_local_field_group([
        'key'      => 'group_bc_hero',
        'title'    => 'BC Hero Section',
        'fields'   => [
            ['key' => 'field_bc_hero_title', 'label' => 'Title', 'name' => 'bc_hero_title', 'type' => 'text', 'required' => 1],
            ['key' => 'field_bc_hero_bg',    'label' => 'Background Image', 'name' => 'bc_hero_background_image', 'type' => 'image', 'return_format' => 'array', 'required' => 1],
            // ... subtitle, description, primaryCTA (group), secondaryCTA (group)
        ],
        'location' => [/* Főoldal page */],
    ]);

    // bc-brand, bc-gallery, bc-services, bc-service,
    // bc-about, bc-team, bc-assistance, bc-contact, bc-map
    // ... mind kódból regisztrálva
});
```

### 9.4. ACF szabályok

| Szabály | Leírás |
|---|---|
| **Code-defined schema** | Minden field group kódból regisztrált (`acf_add_local_field_group`) |
| **GUI ≠ source-of-truth** | Az admin GUI megjeleníti a mezőket, de nem az igazság forrása |
| **Reusable helpers** | Image, CTA, repeater konverzió közös — sp-infra/acf/-ban |
| **Client field groups** | Minden `bc-*` field group a kliens overlay-ban (`sp-benettcar/infra/acf/`) |
| **ACF Free** | Fixed Field Groups → ACF Free elég (nincs Pro, nincs Flexible Content) |

### 9.5. ACF image → Media konverzió

| ACF image field | → | Spektra Media field |
|---|---|---|
| `url` | → | `src` |
| `alt` | → | `alt` (fallback: filename) |
| `width` | → | `width` |
| `height` | → | `height` |
| `sizes.thumbnail` | → | `variants[0].source.url` |
| `sizes.medium` | → | `variants[1].source.url` |
| `sizes.large` | → | `variants[2].source.url` |
| `mime_type` | → | `mimeType` |

---

## 10. Adapter Architecture

### 10.1. createWordPressAdapter (platform — meglévő)

```ts
interface WordPressAdapterConfig {
  apiBase: string
  endpoint?: string   // default: '/wp-json/spektra/v1/site'
  mapResponse: (response: unknown) => SiteData
  auth?: { token: string }
}

function createWordPressAdapter(config: WordPressAdapterConfig): SiteDataAdapter
```

A `mapResponse` dependency injection — a kliens implementálja:
- Platform: nem tudja, hogyan néz ki a WP JSON
- Kliens: tudja, és `SiteData`-vá konvertálja
- Platform: utólag validálja (`validateSiteData`)

### 10.2. createJsonAdapter (platform — meglévő)

```ts
interface JsonAdapterConfig {
  url?: string
  data?: SiteData
}

function createJsonAdapter(config: JsonAdapterConfig): SiteDataAdapter
```

Két mód:
- `data`: inline statikus adat (dev/mock)
- `url`: távoli JSON fetch

### 10.3. Kliens adapter wiring (sp-benettcar)

```ts
// App.tsx — env-alapú adapter switch
const adapter: SiteDataAdapter =
  import.meta.env.VITE_DATA_SOURCE === 'wordpress'
    ? createWordPressAdapter({
        apiBase: import.meta.env.VITE_WP_API_BASE,
        mapResponse: mapWordPressResponse,  // wp-mapper.ts
        auth: import.meta.env.VITE_WP_AUTH_TOKEN
          ? { token: import.meta.env.VITE_WP_AUTH_TOKEN }
          : undefined,
      })
    : createJsonAdapter({ data: siteData })
```

### 10.4. Adapter boundary szabály

```
WordPress API                    Spektra Platform
    │                                 │
    │  raw JSON                       │
    └────────► wp-mapper.ts ──────────┘
              (kliens réteg)     mapResponse → SiteData
                                       │
                                 validateSiteData
                                       │
                                 ✅ érvényes SiteData
```

A `wp-mapper.ts` a **határ** — ez az egyetlen hely, ahol WP-specifikus JSON struktúrát ismer a rendszer.

---

## 11. Validation Pipeline

### 11.1. Három szintű validáció

| Szint | Hely | Mit validál | Ki felelős |
|---|---|---|---|
| **1. Response builder** | PHP (sp-infra plugin) | ACF mezők megléte, null check, type | Infra |
| **2. mapResponse** | TS (sp-benettcar/wp-mapper.ts) | WP JSON → SiteData mapping, normalization | Client |
| **3. validateSiteData** | TS (sp-platform/@spektra/data) | Struktúra érvényesség (site, navigation, pages, sections) | Platform |

### 11.2. Platform validation (validateSiteData) — meglévő

Struktúra-szintű validáció:

- `site` objektum, `site.name` string
- `navigation` objektum, `navigation.primary` array, NavItem-ek
- `pages` array (min 1), page-enként `slug`, `sections`
- Minden section: `id`, `type`, `data` (defined, not null)
- Media: `src` string, `alt` string (ahol media van)
- **NEM** validálja a `section.data` belső szerkezetét

### 11.3. Normalization szabályok (wp-mapper.ts)

| Bemenő adat | → | Normalizált kimenet |
|---|---|---|
| Üres string `""` | → | `undefined` |
| ACF image array | → | `Media` shape |
| Hiányzó optional mező | → | Kihagyás (nem null, nem `""`) |
| Locale hiányzik | → | `"hu-HU"` fallback |
| ACF repeater üres | → | `[]` (üres tömb) |
| ACF group üres | → | `null` → kihagyás |

---

## 12. Section Storage Model

### 12.1. Fixed Field Groups (v1 döntés)

Minden `bc-*` szekció egy fix ACF field group, ami a "Főoldal" WP page-hez van kötve.

```
WordPress → Főoldal page
  ACF field groups:
    group_bc_hero      → bc-hero section data
    group_bc_brand     → bc-brand section data
    group_bc_gallery   → bc-gallery section data
    group_bc_services  → bc-services section data
    group_bc_service   → bc-service section data
    group_bc_about     → bc-about section data
    group_bc_team      → bc-team section data
    group_bc_assistance→ bc-assistance section data
    group_bc_contact   → bc-contact section data
    group_bc_map       → bc-map section data
```

### 12.2. Response builder logika

A `class-response-builder.php` a fix field group-okat olvassa ki és `sections[]`-be pakolja:

```php
$sections = [];
$sections[] = build_hero_section(get_field('bc_hero_title', $page_id), ...);
$sections[] = build_brand_section(...);
// ... mind a 10
// null/üres szekciók kiszűrése
$sections = array_filter($sections);
```

### 12.3. Multi-page ready (jövőbeli)

A v1 single page, de a kód kész a bővítésre:

```json
{
  "pages": [
    { "slug": "home", "sections": [...] },
    { "slug": "about", "sections": [...] },
    { "slug": "services", "sections": [...] }
  ]
}
```

A response builder `pages[]` tömböt ad vissza — nem single-page shortcut.

---

## 13. Seed Strategy

### 13.1. Pipeline

```
site.ts (kliens statikus adat)
  → export-seed.ts (Node script)
    → seed.json (generált, gitignored)
      → WP-CLI / import-seed.sh
        → WordPress DB (ACF field values)
```

### 13.2. Export (site.ts → seed.json)

> **P8.5 boundary döntés**: az export a kliens repóban él, nem sp-infra-ban.
> Lásd: [content-parity-bootstrap.md](content-parity-bootstrap.md) §3.2.

`<client>/infra/seed/export-seed.ts`:
- Importálja a kliens `site.ts`-t
- Kliens-specifikus `mapping.ts`-t használja
- Kimenete: `seed.json` — WP-CLI kompatibilis

### 13.3. Import (seed.json → WP)

`sp-infra/seed/import-seed.sh` (vagy `.ps1`):
- WP-CLI `wp post meta update` hívások
- ACF field value-k beírása
- Idempotens — többször futtatható

### 13.4. Seed szabályok

| Szabály | Leírás |
|---|---|
| `seed.json` gitignored | Generált fájl — nem commitolható |
| Idempotens import | Többször futtatva nem duplikál |
| Kliens-specifikus mapping | A seed export tudja a kliens field name-jeit |
| Debug-barát | Human-readable JSON output |

---

## 14. Expandability Strategy

### 14.1. Új kliens hozzáadása

```
1. Új repo: sp-client-x/
2. infra/ overlay létrehozása (config.php, acf/, env/)
3. WP runtime: .local/wp-runtimes/client-x/
4. Plugin symlink: sp-infra/plugin/spektra-api/ → runtime
5. Client overlay symlink: sp-client-x/infra/ → runtime
6. Saját wp-mapper.ts
7. Saját site.ts (dev fallback)
8. Kész — sp-platform és sp-infra változatlan
```

### 14.2. Új tartalomforrás hozzáadása

```
1. Új adapter factory a platformban (createStripeAdapter, createSanityAdapter, stb.)
   VAGY kliens-szintű custom adapter
2. mapResponse implementáció a kliensben
3. .env switch bővítés
4. sp-infra nem érintett (WP-specifikus)
```

### 14.3. Új section type hozzáadása

```
1. Kliens-szintű: bc-newtype section component + data type
2. ACF overlay: sp-benettcar/infra/acf/sections/bc-newtype.php
3. Response builder: új section assembly logika
4. wp-mapper.ts: új section mapping
5. Platform: NINCS VÁLTOZÁS
```

### 14.4. Új page hozzáadása

```
1. WP: Új page létrehozás + ACF field group location bővítés
2. Response builder: pages[] tömbbe az új page is bekerül
3. wp-mapper.ts: már kezeli a pages[] tömböt
4. Frontend: router / page rendering
5. Platform: NINCS VÁLTOZÁS
```

---

## 15. Naming Conventions

### 15.1. Fogalom-szótár

| Fogalom | Jelentés | Példa |
|---|---|---|
| **apps** | Futtatható infra preset-ek — helyi konfiguráció, indítási receptek | `sp-infra/apps/wamp/` |
| **templates** | Fájl-szintű sablonok — másolandó kiindulópontok | `sp-infra/apps/wamp/vhost.conf` |
| **infra** (sp-infra) | Reusable, kliens-független infra eszközök | `sp-infra/plugin/`, `sp-infra/acf/` |
| **infra** (client overlay) | Kliens-specifikus infra konfiguráció a kliens repo-ban | `sp-benettcar/infra/` |
| **plugin** | A `spektra-api` WP plugin reusable magja | `sp-infra/plugin/spektra-api/` |
| **runtime** | Assembled, gitignored WP példány | `.local/wp-runtimes/benettcar/` |
| **overlay** | Kliens config, ami a runtime-ra "rárétegeződik" | `sp-benettcar/infra/` |
| **seed** | Tartalomgeneráló pipeline (site.ts → WP) | `sp-infra/seed/` |
| **mapper** | Kliens-szintű WP → SiteData konverter | `wp-mapper.ts` |
| **adapter** | Platform-szintű content source interface | `SiteDataAdapter`, `createWordPressAdapter` |

### 15.2. Repository elnevezés

| Pattern | Jelentés | Példa |
|---|---|---|
| `sp-platform` | Spektra platform core | egyetlen repo |
| `sp-infra` | Spektra shared infra | egyetlen repo |
| `sp-{client}` | Kliens repo | `sp-benettcar`, `sp-clientx` |
| `sp-docs` | Dokumentáció | egyetlen repo |

### 15.3. WP mező elnevezés

| Pattern | Példa | Leírás |
|---|---|---|
| `{client}_{section}_{field}` | `bc_hero_title` | Section field |
| `{client}_{section}_{subfield}` | `bc_hero_primary_cta` | Nested field (group) |
| `group_{client}_{section}` | `group_bc_hero` | ACF field group key |
| `field_{client}_{section}_{field}` | `field_bc_hero_title` | ACF field key |

---

## 16. Multi-Client Future Model

### 16.1. Cél-állapot

```
sp-platform/               # 1 db, közös
sp-infra/                   # 1 db, közös
sp-benettcar/               # Client A
sp-client-x/                # Client B
sp-client-y/                # Client C
.local/wp-runtimes/
  benettcar/                # Client A runtime
  client-x/                 # Client B runtime
  client-y/                 # Client C runtime
```

### 16.2. Megosztott vs kliens-specifikus

| Elem | Megosztott (sp-infra) | Kliens-specifikus (client/infra/) |
|---|---|---|
| Plugin PHP | `spektra-api/` | — |
| REST controller | `class-rest-controller.php` | — |
| CORS handler | `class-cors.php` | — |
| Response builder | `class-response-builder.php` (base) | section assembly override (jövő) |
| ACF helpers | `acf/helpers.php` | `acf/field-groups.php` |
| Docker config | `docker/docker-compose.yml` | `env/.env.example` |
| Seed tools | `seed/export-seed.ts` | seed mapping config |
| Bootstrap script | `scripts/bootstrap.ps1` | — |

### 16.3. Skálázhatósági garanciák

| Garancia | Leírás |
|---|---|
| **Új kliens = 0 platform módosítás** | sp-platform nem változik |
| **Új kliens = 0 sp-infra módosítás** | sp-infra nem változik (hacsak nem kell új generikus feature) |
| **Kliens izolált** | Egy kliens módosítása nem hat másikra |
| **Közös frissítés** | sp-infra plugin frissítés → minden kliens kapja (symlink) |
| **Runtime izolált** | Minden kliensnek saját WP runtime (saját DB, saját uploads) |

---

## 17. Guardrails

### 17.1. Platform guardrails

| # | Szabály | Ellenőrzés |
|---|---|---|
| 1 | Platform-ban nincs `wordpress`, `acf`, `wp-` import | Grep check |
| 2 | Platform-ban nincs kliens név (`benettcar`, `bc-*`) | Grep check |
| 3 | `SiteData` a kanonikus contract — nincs alternatív shape | Code review |
| 4 | Új típus hozzáadás csak a Platform Change Policy szerint | Decision record |
| 5 | `validateSiteData` nem validálja section.data belsejét | Unit test |

### 17.2. Client guardrails

| # | Szabály | Ellenőrzés |
|---|---|---|
| 1 | `wp-mapper.ts` az egyetlen WP-ismeret a frontend-ben | Import audit |
| 2 | Section komponensek nem tudnak WP-ről | Code review |
| 3 | `.env` switch — soha nem hardcode-olt adapter | Code review |
| 4 | `site.ts` megmarad dev fallback-ként | File exists check |

### 17.3. Infra guardrails

| # | Szabály | Ellenőrzés |
|---|---|---|
| 1 | sp-infra-ban nincs kliens név | Grep check |
| 2 | Plugin config kívülről jön (nem hardcode) | Code review |
| 3 | Runtime gitignored | `.gitignore` check |
| 4 | ACF field groups kódból regisztráltak | No JSON import, no GUI export |
| 5 | Seed idempotens | Script review |

---

## 18. Risks

### 18.1. Kockázati mátrix

| # | Kockázat | Valószínűség | Hatás | Mitigáció |
|---|---|---|---|---|
| 1 | **ACF Free limitáció** — hiányzó repeater/flexible content | Alacsony | Magas | ACF Free repeater-t támogat; Flexible Content → Pro upgrade path dokumentálva |
| 2 | **Symlink problémák Windows-on** | Közepes | Közepes | Admin PowerShell, `New-Item -ItemType SymbolicLink` tesztelve |
| 3 | **CORS debug nehézség** | Közepes | Alacsony | Dedikált `class-cors.php`, smoke test a P5.5-ben |
| 4 | **Response builder karbantartás** | Közepes | Közepes | Section assembly moduláris (section-enként külön metódus) |
| 5 | **Multi-client config conflict** | Alacsony | Magas | Client overlay → kliens repo-ban, nincs shared state |
| 6 | **WAMP verzió eltérés** | Közepes | Alacsony | PHP 8.3+ minimum, dokumentált WAMP config |
| 7 | **Seed import failure** | Alacsony | Alacsony | WP-CLI idempotens, rollback = delete + reimport |
| 8 | **Platform validation túl szigorú** | Közepes | Közepes | Platform Change Policy — módosítható ha indokolt |

### 18.2. Architekturális kockázatok

| Kockázat | Mitigáció |
|---|---|
| **Response builder monolith** | Moduláris section assembly — minden section külön metódus |
| **Client overlay szétszóródás** | Konvenció + új kliens template (P11.2) |
| **Runtime instabilitás** | Bootstrap script + dokumentált reprodukálási lépések |
| **Mapper complexity** | Normalization külon függvények, unit-tesztelhető |

---

## 19. Migration Strategy v3 → v4

### 19.1. Mi változik

| v3 elem | v4 változás | Akció |
|---|---|---|
| `sp-infra/sp-benettcar/` egyetlen mappa | sp-infra (shared) + sp-benettcar/infra/ (overlay) | SPLIT |
| `sp-infra/sp-benettcar/plugin/` | → `sp-infra/plugin/` | MOVE |
| `sp-infra/sp-benettcar/config/` | → `sp-benettcar/infra/` | MOVE |
| `sp-infra/sp-benettcar/docker/` | → `sp-infra/docker/` | MOVE |
| `sp-infra/sp-benettcar/seed/` | → `sp-infra/seed/` | MOVE |
| `sp-infra/sp-benettcar/wp/` | → `.local/wp-runtimes/benettcar/` | MOVE + GITIGNORE |
| ACF field-ek egy helyen | → `sp-infra/acf/` (helpers) + `sp-benettcar/infra/acf/` (client) | SPLIT |
| Nincs formális tooling | → `sp-infra/scripts/` (bootstrap, symlink, env) | NEW |
| Nincs naming konvenció | → Naming Conventions szekció | NEW |
| Multi-client említve | → Multi-Client Future Model szekció formalizálva | ELEVATE |

### 19.2. Mi NEM változik

- SiteData contract → változatlan
- Media shape → változatlan
- Adapter interface → változatlan
- createWordPressAdapter → változatlan
- validateSiteData → változatlan
- Section payload policy → változatlan
- Auth strategy → változatlan
- Plugin PHP struktúra → változatlan (hely változik, kód nem)
- Seed strategy koncepció → változatlan (hely változik)

### 19.3. Migrációs sorrend

```
1. sp-infra repo létrehozás (üres)
2. sp-infra scaffold (plugin/, acf/, docker/, seed/, scripts/, apps/, docs/)
3. sp-benettcar/infra/ létrehozás
4. v3 plugin/ → sp-infra/plugin/ (ha van már kód)
5. v3 config/ → sp-benettcar/infra/
6. v3 docker/ → sp-infra/docker/
7. v3 seed/ → sp-infra/seed/
8. .local/wp-runtimes/ setup
9. Symlink-ek konfigurálása
10. v3 docs frissítés → v4 referenciák
```

---

## 20. TODO Roadmap

### Phase 0 — Architecture Baseline

**Cél**: A v4 architektúra dokumentálása, döntések véglegesítése, v3→v4 migration mapping.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P0.1 | v4 baseline doc megírása (20 szekció) | `sp-docs/wp-integration-plan-v4.md` | Minden szekció kitöltve, v3 tartalom megőrizve |
| P0.2 | v3→v4 migration mapping | Migration szekció a v4 doc-ban | Minden v3 elem → v4 hely leképezve |
| P0.3 | Decision records véglegesítés | Döntések listája a v4 doc-ban | Fixed Field Groups, WAMP-first, ACF Free, overlay model rögzítve |

---

### Phase 1 — Repository Setup

**Cél**: sp-infra repo és sp-benettcar/infra/ overlay létrehozása, boundary szabályok definiálása.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P1.1 | sp-infra repo létrehozás | `D:\Projects\spektra\sp-infra\` + GitHub repo | Repo elérhető, README.md van |
| P1.2 | sp-infra alapstruktúra scaffold | `plugin/`, `acf/`, `docker/`, `seed/`, `scripts/`, `apps/`, `docs/` mappák | Minden mappa létezik, README-kkel |
| P1.3 | sp-benettcar/infra/ overlay scaffold | `infra/config.php`, `infra/acf/`, `infra/env/`, `infra/docs/` | Overlay mappa struktúra kész |
| P1.4 | .gitignore + boundary rules | `.local/` gitignored, runtime nem commitolható | `git status` clean runtime nélkül |

---

### Phase 2 — sp-infra Bootstrap

**Cél**: Az sp-infra reusable elemek alapvázának felépítése.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P2.1 | plugin/ base skeleton | `spektra-api.php` + `includes/` üres fájlokkal | Plugin struktúra kész, nincs kód |
| P2.2 | acf/ reusable helpers scaffold | `helpers.php`, `media-helper.php` üres | Fájlok léteznek, require nem hibázik |
| P2.3 | docker/ base config | `docker-compose.yml` + `.env.example` | `docker-compose config` hibamentes |
| P2.4 | seed/ + scripts/ + docs/ scaffold | Mappa struktúra + placeholder README-k | Minden mappa létezik |
| P2.5 | Tooling & Dev Experience | `bootstrap.ps1`, `link-plugin.ps1`, `link-overlay.ps1`, `setup-env.ps1` | Script-ek futtathatók (még üres logikával) |

---

### Phase 3 — Client Infra Overlay

**Cél**: A Benettcar kliens-specifikus infra konfigurációjának elkészítése.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P3.1 | sp-benettcar/infra/config.php | CORS origins, site defaults, plugin config | PHP szintaktikailag helyes, `php -l` OK |
| P3.2 | Content model spec (site.ts ↔ WP) | Mezőtérkép: TS field → WP field (10 section) | Minden required/optional mező leképezve |
| P3.3 | sp-benettcar/infra/acf/ field groups | 10 db `bc-*` field group PHP definíció | Szintaktikailag helyes PHP |
| P3.4 | .env.example + overlay docs | `infra/env/.env.example` + `infra/docs/` | Dokumentáció olvasható |

---

### Phase 4 — WordPress Runtime Assembly ⚙

> **Hard Architecture** — Ez dönti el: deploy modell, local modell, multi-client modell.

**Cél**: Működő WordPress runtime összeállítása WAMP-on, symlink-ekkel.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P4.1 | WAMP vhost (benettcar.local) | `httpd-vhosts.conf` + hosts fájl | `http://benettcar.local` válaszol |
| P4.2 | WordPress install + alap config | WP telepítve, permalink, nyelv, front page | WP admin elérhető, "Főoldal" page létezik |
| P4.3 | Plugin symlink (sp-infra → runtime) | `wp-content/plugins/spektra-api → sp-infra/plugin/spektra-api` | Plugin látható WP admin-ban |
| P4.4 | Client overlay → runtime linking | `wp-content/plugins/spektra-config → sp-benettcar/infra` | Config fájlok elérhetők a runtime-ból |
| P4.5 | ACF install + activate + verify | ACF plugin telepítve, `acf_add_local_field_group` elérhető | `function_exists('acf_add_local_field_group')` → true |

---

### Phase 5 — Plugin Architecture

**Cél**: A `spektra-api` plugin működő REST API-t szolgáltat.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P5.1 | spektra-api.php (header, hooks) | Plugin bootstrap, autoload, hook registration | Plugin aktiválható WP admin-ban |
| P5.2 | class-rest-controller.php | `register_rest_route('spektra/v1', '/site', [...])` | Route regisztrálva, 200 OK |
| P5.3 | class-cors.php (CORS + preflight) | OPTIONS 204, Allow-Origin, Allow-Headers | Preflight OK Vite origin-ről |
| P5.4 | Config loading (core → client) | Plugin betölti `sp-benettcar/infra/config.php`-t | Config értékek elérhetők a plugin-ban |
| P5.5 | REST + CORS smoke test | `curl -i http://benettcar.local/wp-json/spektra/v1/site` | HTTP 200, JSON, CORS header OK |

---

### Phase 6 — ACF Architecture

**Cél**: ACF field group-ok regisztrálva és működnek.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P6.1 | Reusable ACF helpers (sp-infra/acf/) | `helpers.php`, `media-helper.php` implementálva | Require OK, függvények hívhatók |
| P6.2 | Client field registration (10 bc-*) | Minden field group regisztrálva kódból | WP admin: "Főoldal" szerkesztőn 10 csoport megjelenik |
| P6.3 | ACF admin verification | Mező-típusok, repeater-ek, image mezők tesztje | Minden mező funkcionális az admin-ban |
| P6.4 | Teszt adat + persistence check | Teszt értékek beírva, mentés, újratöltés | Adat megmarad mentés után |

---

### Phase 7 — Response Builder ⚙

> **Hard Architecture** — Ez dönti el: contract stabilitás, future CMS support, API boundary.

**Cél**: A WP REST endpoint teljes SiteData-kompatibilis JSON-t ad vissza.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P7.1 | response-builder.php skeleton | Osztály váz, section assembly interface | PHP szintaktikailag helyes |
| P7.2 | Site meta + Navigation assembly | `site:{}` + `navigation:{}` összeállítás | JSON output: site + navigation OK |
| P7.3 | Section assembly (ACF → sections[]) | 10 bc-* section → `sections[]` tömb | Minden section típus megjelenik a JSON-ban |
| P7.4 | Media normalization (ACF → Media) | ACF image → `{ src, alt, width, height, variants }` | Media shape pontos (src ≠ url a gyökérnél) |
| P7.5 | Full SiteData endpoint teszt | Teljes JSON a `/wp-json/spektra/v1/site` endpoint-on | `validateSiteData` pass a kapott JSON-nal |

---

### Phase 8 — Frontend Adapter Integration

**Cél**: A Benettcar frontend WP adatforrásról is tud renderelni.

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P8.1 | wp-mapper.ts (mapResponse) | `mapWordPressResponse(raw: unknown): SiteData` | Mapping helyes, type-safe |
| P8.2 | Normalization + validation pipeline | Üres string → undefined, locale fallback, stb. | Edge case-ek kezelve |
| P8.3 | .env + App.tsx adapter switch | `VITE_DATA_SOURCE=wordpress` → WP adapter aktív | Env switch működik |

---

### Phase 8.5 — Content Parity Bootstrap

**Cél**: A WP adatbázis tartalma megegyezik a site.ts tartalommal. Seed pipeline + parity gate.

> Részletes workstream definíció: [content-parity-bootstrap.md](content-parity-bootstrap.md)

| # | Task | Output | Siker-kritérium |
|---|---|---|---|
| P8.5.1 | Architecture Closure | Ownership + boundary döntések dokumentálva | Workstream def kész |
| P8.5.2 | Guardrail Definition | CMS Runtime Parity Gate spec | Parity check szabályok rögzítve |
| P8.5.3 | Tooling Design | seed.json shape, mapping spec | Technikai terv kész |
| P8.5.4 | Minimal Implementation | export-seed.ts, mapping.ts, import-seed.sh | seed.json generálódik, WP import fut |
| P8.5.5 | Verification | verify-parity.ts PASS | WP tartalom = site.ts tartalom |
| P8.5.6 | Documentation | Postmortem, workflow standard | Onboarding checklist kész |

---

### Phase 9 — First Vertical Slice (E2E) ✅ COMPLETED (2026-04-11)

**Cél**: A frontend bizonyítottan WP-ről fut, 3 szekció end-to-end renderel.

| # | Task | Output | Siker-kritérium | Státusz |
|---|---|---|---|---|
| P9.1 | Runtime Cutover Baseline | Frontend WP módban indul | WP adapter aktív, no silent fallback, endpoint fetch OK, CORS alap | ✅ |
| P9.2 | bc-hero E2E (Media + CTA) | Hero section WP-ből renderelve | title/desc/image/CTA render, partial fail-soft | ✅ |
| P9.3 | bc-services E2E (repeater) | Services section WP repeater-ből | Lista renderelve, sorrend stabil, üres eset kezelve | ✅ |
| P9.4 | bc-contact E2E (text fields) | Contact section WP-ből | Szöveges mezők megjelennek, key normalization helyes | ✅ |
| P9.5 | CORS + live update verify | WP módosítás → frontend frissítés | Network-szintű CORS evidence, adat frissül | ✅ |

**P9.1 checkpointok:**

1. `.env` → `VITE_DATA_SOURCE=wordpress` + `VITE_WP_API_BASE=http://benettcar.local`
2. `createWordPressAdapter()` aktiválódik (nem json fallback)
3. **No silent fallback** — fetch hiba → error state, nem json mód
4. Endpoint fetch → `200 OK` + valid JSON
5. CORS — Network panel: response header + preflight + origin (`http://localhost:5173`)

**P9.2 checkpointok:**

1. `title` / `subtitle` / `description` renderel
2. `backgroundImage.src` valid URL, broken image nincs, `alt` eljut a renderig
3. `primaryCTA` / `secondaryCTA` — label renderel, href helyes, anchor element
4. Optional CTA hiánya → nincs crash
5. Hiányzó image → section stabil marad

**P9.5 evidencia:**

- Network panel response header ellenőrzés (nem csak console)
- Preflight + actual request viselkedés
- Origin pontos rögzítése

---

### Phase 10 — Stabilization ✅ COMPLETED (2026-04-11)

**Cél**: Teljes 10 szekciós működés, fallback UX, seed pipeline, debug tooling.

| # | Task | Output | Siker-kritérium | Státusz |
|---|---|---|---|---|
| P10.1 | Loading + error state UX | Loading spinner, WP unavailable error UI | UX tesztelve (gyors net, lassú net, WP down) | ✅ |
| P10.2 | Missing field fallback | Degradált renderelés hiányzó optional mezőknél | Nincs crash | ✅ |
| P10.3 | Maradék 7 bc-* szekció bekötés | bc-brand, bc-gallery, bc-about, bc-team, bc-assistance, bc-service, bc-map | Mind a 10 szekció E2E működik | ✅ |
| P10.4 | ~~Seed pipeline (site.ts → WP)~~ | ~~export-seed.ts → seed.json → WP import~~ | → **Phase 8.5-be áthelyezve** (P8.5.4–P8.5.5) | — |
| P10.5 | Debug tooling + logging | Mapper unknown-section warning, normalizer dropped-section warning (`import.meta.env.DEV` gated) | Dev console-ban látható, prod buildből tree-shaken | ✅ |

---

### Phase 11 — Multi-Client Preparation ✅ COMPLETED (2026-04-11)

> **Hard Architecture** — Ez dönti el: platform sikeresség, skálázhatóság, reuse arány.

**Cél**: Dokumentáció és eszközök, amivel új kliens onboard-olása lehetséges.

| # | Task | Output | Siker-kritérium | Státusz |
|---|---|---|---|---|
| P11.1 | Multi-Client Boundary Audit | `sp-docs/knowledge/phases/phase-11.1.md` | Ownership matrix, 8 finding, bootstrap requirements | ✅ |
| P11.2 | Section Builder Delegation | `sp-infra/acf/sections.php` refactor + `sp-benettcar/infra/acf/builders.php` | Shared infra nem tartalmaz bc-* buildereket; overlay-ból betölthető | ✅ |
| P11.3 | Platform + Infra Naming Refactor | `@spektra/layouts`, preset scaffolding, `apps/playground` | templates→layouts rename, 3 preset, audit F1–F4 fix | ✅ |
| P11.4 | Script Parameterization | `seed-pipeline.ps1 -Client`, data-driven `verify-endpoint.php` | Mindkét script `-Client` paraméterrel fut bármely kliensre | ✅ |
| P11.5 | New Client Onboarding Guide | `sp-docs/knowledge/guides/new-client-onboarding.md` | Preset-based onboarding guide, standalone walkthrough | ✅ |

---

## Appendix A — Teljes típus-hivatkozás

### Platform típusok (@spektra/types)

```ts
// adapter.ts
interface SiteDataAdapter {
  load(): Promise<SiteData>
  init?(): Promise<void>
  revalidate?(): Promise<SiteData>
  onError?(error: unknown): void
}

// site.ts
interface SiteMeta { name: string; description?: string; url?: string; locale?: string }
interface SiteData { site: SiteMeta; navigation: Navigation; pages: Page[] }

// navigation.ts
interface NavItem { label: string; href: string; children?: NavItem[]; external?: boolean }
interface Navigation { primary: NavItem[]; footer?: NavItem[] }

// page.ts
interface PageMeta { title?: string; description?: string; ogImage?: Media; canonical?: string }
interface Page { slug: string; title?: string; meta?: PageMeta; sections: Section[] }

// section.ts
type PlatformSectionType = 'hero' | 'features' | 'about' | 'gallery' | 'contact' | 'faq' | 'cta'
type SectionType<T extends string = never> = PlatformSectionType | T
interface SectionMeta { label: string; category: string; description?: string }
interface Section<T = unknown> { id: string; type: string; data: T; meta?: SectionMeta }

// media.ts
interface MediaSource { url: string; width?: number; height?: number; format?: string }
interface MediaVariant { name: string; source: MediaSource }
interface Media { src: string; alt: string; width?: number; height?: number; variants?: MediaVariant[]; mimeType?: string }

// cta.ts
interface CallToAction { text: string; href?: string }

// theme.ts
interface ThemeColors { primary: string; secondary: string; accent?: string; background?: string; foreground?: string }
interface ThemeFonts { sans?: string; heading?: string; mono?: string }
interface ThemeConfig { name: string; colors: ThemeColors; fonts?: ThemeFonts }
```

### Platform adapterek (@spektra/data)

```ts
// wordpress.ts
interface WordPressAdapterConfig {
  apiBase: string
  endpoint?: string
  mapResponse: (response: unknown) => SiteData
  auth?: { token: string }
}
function createWordPressAdapter(config: WordPressAdapterConfig): SiteDataAdapter

// json-adapter.ts
interface JsonAdapterConfig { url?: string; data?: SiteData }
function createJsonAdapter(config: JsonAdapterConfig): SiteDataAdapter

// validate.ts
type SiteDataValidationResult = { valid: true; data: SiteData } | { valid: false; errors: string[] }
function validateSiteData(input: unknown): SiteDataValidationResult
```

---

## Appendix B — Decision Records

Minden architekturális döntés egy helyen, indoklással és alternatívákkal.

### DR-001: WordPress = opcionális content source

| | |
|---|---|
| **Döntés** | A WordPress opcionális tartalomforrás, nem a Spektra core identity |
| **Indoklás** | A platform multi-source — WP mellett JSON, Sanity, Strapi, stb. is lehetséges |
| **Alternatíva** | WP-first platform (elvetett — lock-in, nem skálázható) |
| **Következmény** | Platform kódban nincs WP import, nincs ACF referencia, nincs PHP logika |
| **Státusz** | ✅ Végleges |

### DR-002: SiteData mint kanonikus contract

| | |
|---|---|
| **Döntés** | A `SiteData` a platform egyetlen igazságforrása — minden adapter ezt produkálja |
| **Indoklás** | Egységes interface a frontend felé, függetlenül a tartalomforrástól |
| **Alternatíva** | Adapter-specifikus shape-ek (elvetett — frontend komplexitás nő) |
| **Következmény** | Minden új forrás adapter `mapResponse`-t implementál → `SiteData` |
| **Státusz** | ✅ Végleges |

### DR-003: Fixed Field Groups (ACF Free)

| | |
|---|---|
| **Döntés** | Minden `bc-*` szekció fix ACF field group — ACF Free elég |
| **Indoklás** | V1 single landing page, 10 fix section — Flexible Content overhead felesleges |
| **Alternatíva** | ACF Pro + Flexible Content (jövőben lehetséges upgrade path) |
| **Következmény** | Szekció sorrend a PHP kódban fix; tartalom szerkeszthető, struktúra nem |
| **Upgrade path** | Ha dinamikus szekció sorrend kell → ACF Pro + Flexible Content migration |
| **Scope pontosítás** | A "Fixed Field Groups" a szekció/page-szintű fix field group struktúrára vonatkozik. A repeatable content source strategy külön governance alatt áll — lásd DR-015 |
| **ACF Free baseline** | ACF Free marad a baseline cél. Az `acf_add_local_field_group()` mechanizmus nem Pro-specifikus. A Pro dependency a `'type' => 'repeater'` field type-ból ered, mert az ACF Repeater ACF Pro feature |
| **Státusz** | ⚠️ Korrigálva — lásd DR-015 és `concepts/repeatable-content-source-strategy.md` |

### DR-004: WAMP-first, Docker prepared

| | |
|---|---|
| **Döntés** | Az elsődleges local runtime WAMP, Docker előkészített de nem elsődleges |
| **Indoklás** | Aktuális fejlesztői környezet Windows + WAMP, egyszerűbb debug |
| **Alternatíva** | Docker-first (elvetett — overhead, Windows perf issues) |
| **Következmény** | WAMP setup dokumentálva, Docker `docker-compose.yml` kész de nem elsődleges |
| **Státusz** | ✅ Végleges |

### DR-005: sp-infra = shared repo, client overlay = kliens repoban

| | |
|---|---|
| **Döntés** | Reusable infra (plugin, ACF helpers, docker, seed) → `sp-infra` shared repo. Kliens-specifikus config → `sp-benettcar/infra/` |
| **Indoklás** | Elkerüli a kód duplikációt (plugin minden kliensben ugyanaz), de a kliens config izolált marad |
| **Alternatíva** | A: Per-client sp-infra repo (elvetett — duplikáció). B: Mindent sp-infra-ban (elvetett — kliens szennyeződés) |
| **Következmény** | Plugin symlink a runtime-ba, client overlay symlink külön |
| **Státusz** | ✅ Végleges |

### DR-006: WP runtime = assembled, gitignored

| | |
|---|---|
| **Döntés** | A WordPress runtime `.local/wp-runtimes/{client}/` alatt él, gitignored, script-tel összeállítva |
| **Indoklás** | WP core fájlok nem commitolhatók, runtime = assembled state, nem source-of-truth |
| **Alternatíva** | WP a repo-ban (elvetett — git bloat, security risk) |
| **Következmény** | Bootstrap script szükséges, runtime bármikor újraépíthető |
| **Státusz** | ✅ Végleges |

### DR-007: Code-defined ACF schema (GUI ≠ source-of-truth)

| | |
|---|---|
| **Döntés** | Minden ACF field group PHP kódból regisztrálva (`acf_add_local_field_group`) |
| **Indoklás** | Verziózható, reprodukálható, review-olható — GUI-ból exportált JSON nem megbízható |
| **Alternatíva** | ACF JSON sync (elvetett — merge conflict, nem auditálható) |
| **Következmény** | Admin GUI olvasható (megjeleníti a mezőket), de nem szerkeszthető source |
| **Státusz** | ✅ Végleges |

### DR-008: Adapter boundary = wp-mapper.ts

| | |
|---|---|
| **Döntés** | A `wp-mapper.ts` (kliens réteg) az egyetlen hely, ahol WP JSON struktúrát ismer a frontend |
| **Indoklás** | Egyetlen fájl cseréje / módosítása elég ha WP response shape változik |
| **Alternatíva** | WP-aware komponensek (elvetett — tight coupling) |
| **Következmény** | Section komponensek `SiteData`-t kapnak, nem WP JSON-t |
| **Státusz** | ✅ Végleges |

### DR-009: Public REST endpoint, Bearer token preview-hoz

| | |
|---|---|
| **Döntés** | `GET /wp-json/spektra/v1/site` publikus (nincs auth). Preview mód: `?preview=true` + Bearer token |
| **Indoklás** | Published tartalom publikus — nincs szükség auth-ra. Draft/preview védett |
| **Alternatíva** | Minden request auth-os (elvetett — felesleges overhead publikus tartalomra) |
| **Következmény** | CORS szükséges (frontend más origin-ről kér), OPTIONS preflight kezelendő |
| **Státusz** | ✅ Végleges |

### DR-010: Multi-page ready, single page v1

| | |
|---|---|
| **Döntés** | Response shape: `pages[]` → `sections[]`. V1-ben egyetlen "Főoldal" page |
| **Indoklás** | A kód most is tömb-alapú — nem kell refactor ha új page jön |
| **Alternatíva** | Single-page shortcut (elvetett — later refactoring cost) |
| **Következmény** | Response builder `pages[]`-t ad vissza, soha nem flat sections |
| **Státusz** | ✅ Végleges |

### DR-011: Naming convention — apps / templates / overlay

| | |
|---|---|
| **Döntés** | `apps` = futtatható infra preset-ek, `templates` = fájl-szintű sablonok, `infra` (kliens) = client overlay |
| **Indoklás** | Egyértelmű fogalom-szétválasztás — nincs "config" vs "template" keveredés |
| **Alternatíva** | Mindent `config/`-nak hívni (elvetett — ambiguity) |
| **Következmény** | `sp-infra/apps/` = preset-ek, `sp-benettcar/infra/` = overlay |
| **Státusz** | ✅ Végleges |

### DR-012: Kétszintű ACF modell (reusable helpers + client groups)

| | |
|---|---|
| **Döntés** | ACF helpers (image→Media, CTA, repeater) → `sp-infra/acf/`. Client field groups → `sp-benettcar/infra/acf/` |
| **Indoklás** | Image→Media konverzió minden kliensben ugyanaz, de a field group definíció kliens-specifikus |
| **Alternatíva** | Mindent a kliens overlay-ban (elvetett — duplikáció új klienseknél) |
| **Következmény** | Új kliens: saját field groups, de meglévő helper-ek újrahasználhatók |
| **Státusz** | ✅ Végleges |

### DR-013: Seed pipeline — site.ts → seed.json → WP-CLI

| | |
|---|---|
| **Döntés** | A kliens `site.ts` statikus adatából generálunk `seed.json`-t, amit WP-CLI-vel importálunk |
| **Indoklás** | A `site.ts` már tartalmazza a teljes tartalmat — ebből automatikusan feltölthető a WP |
| **Alternatíva** | Kézi tartalombevitel WP admin-ban (elvetett — lassú, hibára hajlamos) |
| **Következmény** | `export-seed.ts` ismeri a kliens field name mapping-et |
| **Státusz** | ✅ Végleges |

### DR-014: Platform Change Policy érvényes

| | |
|---|---|
| **Döntés** | A WP integráció során a platform módosítható, ha javítja az architektúrát |
| **Indoklás** | A platform még építés alatt — ha workaround kell a kliensben, jobb a platformot javítani |
| **Korlát** | WP-specifikus logika és kliens-specifikus mezők SOHA nem kerülnek a platformba |
| **Következmény** | Adapter contract, Media shape, validation lazítható — de a boundary megmarad |
| **Státusz** | ✅ Végleges |

---

### DR-015: Repeatable Content Source Strategy

| | |
|---|---|
| **Döntés** | Repeatable content háromféle source strategy-n keresztül kezelt: `cpt_collection` (default free), `fixed_slots` (bounded), `acf_repeater_optional` (opcionális Pro) |
| **Indoklás** | DR-003 ACF Free sufficiency-t állított, de mindkét kliens inception óta ACF Pro repeater-t használ (12 field, 10 szekció). A free baseline helyreállítása source strategy absztrakcióval történik |
| **Alternatíva** | ACF Pro mint baseline requirement elfogadása — elutasítva, mert sérti a free/open-source pozícionálást |
| **Következmény** | Meglévő repeater field-ek classification szükséges; új repeater field tilos a free baseline-ban; SiteData shape változatlan marad |
| **SiteData hatás** | Nincs — source strategy csere a builder mögött történik, frontend contract stabil |
| **Cross-reference** | `concepts/repeatable-content-source-strategy.md`, `phases/phase-12.2.md`, DR-003 (korrigálva) |
| **Státusz** | 📋 Proposed |
