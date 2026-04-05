# Spektra REST API — Architektúra, működés, kontextus

> Célközönség: junior fejlesztők, senior fejlesztők, külső kontribútorok.
> Utolsó frissítés: 2026-04-05
> **Implementációs státusz: aktív fejlesztés alatt** (Phase 5 — lásd [Aktuális állapot](#aktuális-állapot))

---

## Tartalomjegyzék

1. [Mi az a REST API?](#mi-az-a-rest-api)
2. [Miért kell API a Spektrának?](#miért-kell-api-a-spektrának)
3. [Az architektúra nagy képe](#az-architektúra-nagy-képe)
4. [A plugin felépítése](#a-plugin-felépítése)
5. [Az endpoint: /spektra/v1/site](#az-endpoint-spektrav1site)
6. [SiteData — a válasz formátuma](#sitedata--a-válasz-formátuma)
7. [Kliens config és overlay rendszer](#kliens-config-és-overlay-rendszer)
8. [CORS — cross-origin hozzáférés](#cors--cross-origin-hozzáférés)
9. [A kérés útja — teljes kép](#a-kérés-útja--teljes-kép)
10. [WordPress REST API — miért erre építünk](#wordpress-rest-api--miért-erre-építünk)
11. [Mi különbözteti meg a Spektra API-t](#mi-különbözteti-meg-a-spektra-api-t)
12. [Aktuális állapot](#aktuális-állapot)
13. [Összefoglaló](#összefoglaló)

---

## Mi az a REST API?

### A legegyszerűbb magyarázat

Egy REST API egy **cím (URL)**, amire ha ráküldessz egy kérést, kapsz egy választ.

```
Kérés:   GET http://benettcar.local/wp-json/spektra/v1/site
Válasz:  { "site": { "name": "Benett Car" }, "navigation": [...], "pages": [...] }
```

Ugyanaz az elv, mint amikor a böngészőbe beírsz egy URL-t — csak itt nem HTML oldalt
kapsz vissza, hanem **nyers adatot** (JSON formátumban), amit egy program dolgoz fel.

### REST = szabályrendszer

A REST (Representational State Transfer) nem egy technológia, hanem egy **szabálygyűjtemény**,
ami megmondja, hogyan szervezd az URL-jeidet és mit jelentsenek a HTTP metódusok:

| HTTP metódus | Jelentés | Példa |
|---|---|---|
| `GET` | Adatot kérek | `GET /site` → add a site adatait |
| `POST` | Újat hozok létre | `POST /pages` → hozz létre egy új oldalt |
| `PUT` | Felülírom az egészet | `PUT /pages/1` → cseréld ki az 1-es oldalt |
| `PATCH` | Részben módosítom | `PATCH /pages/1` → módosítsd a címet |
| `DELETE` | Törlöm | `DELETE /pages/1` → töröld az 1-es oldalt |

A Spektra API jelenleg **csak GET-et** használ — adatot szolgáltat, nem módosít.

### API = szerződés

Az API (Application Programming Interface) lényege a **szerződés** a két oldal között:

- **Szerver** (WordPress + Spektra plugin): „Ha erre a címre küldessz GET-et, ezt a
  formátumú JSON-t kapod vissza."
- **Kliens** (Vite + React frontend): „Rendben, erre a formátumra építem a megjelenítést."

Ha bármelyik oldal megszegi a szerződést — a szerver más formátumot küld, vagy a kliens
mást vár — **elromlik a rendszer**. Ezért fontos, hogy a formátum (schema) pontosan
definiált legyen.

---

## Miért kell API a Spektrának?

### A probléma

A Spektra egy **headless** rendszer: a tartalom és a megjelenítés szét van választva.

```
┌──────────────────────┐        ┌──────────────────────┐
│  WordPress (tartalom) │        │  React app (UI)      │
│  - ACF mezők          │        │  - komponensek       │
│  - képek, szövegek    │        │  - CSS, animáció     │
│  - admin felület      │        │  - routing           │
└──────────┬───────────┘        └──────────┬───────────┘
           │                               │
           │    nincs közvetlen kapcsolat   │
           └───────────── ??? ─────────────┘
```

A WordPress és a React app **két különálló alkalmazás**. Különböző szerveren futhatnak,
különböző nyelven vannak írva (PHP vs. TypeScript), különböző porton hallgatnak. Nincs
közös memória, nincs közös adatbázis, nincs közös fájlrendszer.

### A megoldás

A REST API a **híd** a két oldal között:

```
┌──────────────────────┐        ┌──────────────────────┐
│  WordPress            │        │  React app            │
│  + Spektra API plugin │◄──────►│  + SiteDataAdapter    │
│  + ACF                │  JSON  │  + fetch()            │
└──────────────────────┘        └──────────────────────┘
        :80 (Apache)                 :5174 (Vite)
```

A plugin kiszedi a tartalmat a WordPress-ből (ACF mezők, site beállítások),
**JSON-ba csomagolja**, és kiadja egy URL-en. A React app lekéri és megjeleníti.

### Miért nem más megoldás?

| Alternatíva | Miért nem |
|---|---|
| **WordPress template-ek** (PHP render) | Nincs React, nincs komponens-újrahasználás, nincs SPA |
| **GraphQL** (pl. WPGraphQL) | Over-engineering egy CMS-re, ami fix sémát szolgáltat |
| **Direkt DB lekérés** | Biztonsági rémálom, nem skálázódik, platform-függő |
| **Statikus export** | Nem real-time, nem preview-képes, build kell minden változtatáshoz |
| **WP REST API natív** | Van, de a SiteData shape nem egyezik — wrapper kell |

A Spektra választása: **egyetlen, custom REST endpoint**, ami pontosan a frontend által
várt formátumot (`SiteData`) adja vissza. Nincs adapter réteg a kliensben, nincs
transzformáció — amit a szerver küld, azt a React rendereli.

---

## Az architektúra nagy képe

```
                         benettcar.local
                              │
                    ┌─────────┴──────────┐
                    │  Apache (WAMP)     │
                    │  :80               │
                    └─────────┬──────────┘
                              │
               /wp-json/spektra/v1/site
                              │
                    ┌─────────┴────────────────────────┐
                    │  WordPress 6.9.4                 │
                    │  ┌────────────────────────────┐  │
                    │  │  spektra-api plugin        │  │
                    │  │  ├── Rest_Controller       │  │ ← route regisztráció
                    │  │  ├── CORS                  │  │ ← cross-origin headerek
                    │  │  └── Response_Builder      │  │ ← JSON összeállítás
                    │  └──────────┬─────────────────┘  │
                    │             │                    │
                    │  ┌──────────┴─────────────────┐  │
                    │  │  spektra-config (Junction) │  │
                    │  │  ├── config.php            │  │ ← kliens beállítások
                    │  │  └── acf/field-groups.php  │  │ ← ACF mező definíciók
                    │  └────────────────────────────┘  │
                    │                                  │
                    │  ┌────────────────────────────┐  │
                    │  │  ACF Free                  │  │ ← tartalomkezelés
                    │  └────────────────────────────┘  │
                    └──────────────────────────────────┘
                              │
                         JSON válasz
                              │
                    ┌─────────┴──────────┐
                    │  React app         │
                    │  localhost:5174    │
                    └────────────────────┘
```

Négy repo vesz részt:

| Repo | Szerep | Példa tartalom |
|---|---|---|
| **sp-infra** | Plugin forráskód (reusable) | `spektra-api.php`, `class-rest-controller.php` |
| **sp-benettcar** | Kliens config overlay | `config.php`, `acf/field-groups.php` |
| **sp-platform** | Frontend app (React + Vite) | `App.tsx`, adapter, runtime |
| **sp-engine** | Közös típusok (TypeScript) | `SiteData.ts` — a szerződés TypeScript oldala |

---

## A plugin felépítése

A plugin 4 fájlból áll + a kliens overlay:

```
wp-content/plugins/
├── spektra-api/        ← Junction → sp-infra/plugin/spektra-api/
│   ├── spektra-api.php           ← belépési pont, constants, autoload, config
│   └── includes/
│       ├── class-rest-controller.php  ← route regisztráció
│       ├── class-cors.php             ← CORS kezelés
│       └── class-response-builder.php ← JSON összeállítás
│
└── spektra-config/     ← Junction → sp-benettcar/infra/
    ├── config.php                ← kliens beállítások
    └── acf/
        └── field-groups.php      ← ACF mező definíciók
```

### spektra-api.php — a belépési pont

Ez a fájl **nem csinál üzleti logikát**. Négy dolgot végez:

1. **Konstansok definiálása** — `SPEKTRA_API_VERSION`, `SPEKTRA_API_PATH`, `SPEKTRA_API_URL`
2. **Includes betöltése** — `require_once` a három osztályra
3. **Kliens config betöltése** — `config.php` beolvasása, `SPEKTRA_CLIENT_CONFIG` konstansba
4. **Hook regisztráció** — `rest_api_init`-ra felakasztja a Controller-t és a CORS-t

Fontos elv: a plugin **soha nem tartalmaz kliens-specifikus kódot**. Amit a kliens
testreszab, az a `spektra-config/` overlay-ben van.

### class-rest-controller.php — a route

Egyetlen REST route-ot regisztrál:

```
GET /wp-json/spektra/v1/site
```

A `register_routes()` a WordPress `register_rest_route()` függvényét hívja.
A `handle_request()` fogadja a kérést és delegálja a Response_Builder-nek.

### class-cors.php — a CORS kezelő

A frontend és a backend különböző porton/domenen fut. A böngésző ilyenkor blokkolja
a kérést, hacsak a szerver nem küld **CORS headereket** (erről bővebben [lentebb](#cors--cross-origin-hozzáférés)).

### class-response-builder.php — a JSON gyár

Ez az osztály szedi össze a WordPress-ből (ACF mezők, site beállítások) az adatot, és
a `SiteData` formátumba csomagolja. A plugin lényegének 80%-a itt fog élni.

---

## Az endpoint: /spektra/v1/site

### URL felépítés

```
http://benettcar.local/wp-json/spektra/v1/site
└──────────────────┘ └──────┘ └────────┘ └──┘
     domain           WP REST   namespace  route
                      prefix
```

| Rész | Érték | Miért |
|---|---|---|
| Domain | `benettcar.local` | Kliens-specifikus vhost (WAMP) |
| WP REST prefix | `/wp-json/` | WordPress beépített REST alap URL |
| Namespace | `spektra/v1` | API verziókezelés — ha a formátum változik, jön a `v2` |
| Route | `/site` | Egyetlen route — a teljes site adat |

### Miért egyetlen endpoint?

A Spektra nem egy általános CMS API (mint a WP natív REST), ahol az összes post type-nak
saját endpoint-ja van. A Spektra **egyetlen kliens-oldalt** szolgál ki:

```
/site             → teljes SiteData (site meta + navigation + pages + sections)
/site?preview=true → ugyanaz, de draft/preview tartalommal
```

Egyetlen endpoint előnyei:
- **Egy kérés = egy oldalbetöltés** — nincs waterfall (fetch site → fetch nav → fetch sections)
- **A formátum a frontend által várt shape** — nincs kliens-oldali transzformáció
- **Egyszerű cache invalidation** — egy URL-t kell invalidálni, nem tízet

### Paraméterek

| Param | Típus | Alap | Cél |
|---|---|---|---|
| `preview` | `string` (`"true"`) | nincs | Draft tartalom is megjelenik |

Jelenleg ez az egyetlen paraméter. Senere bővülhet (pl. `?section=hero`, `?lang=en`).

---

## SiteData — a válasz formátuma

A SiteData a **szerződés** a szerver és a kliens között. TypeScript-ben van definiálva
(`sp-engine/src/types/SiteData.ts`), a PHP oldal ezt a shape-et reprodukálja:

### Az interfész

```typescript
interface SiteData {
  site: SiteMeta
  navigation?: Navigation | NavItem[]
  pages: Page[]
}

interface SiteMeta {
  name: string         // "Benett Car"
  description?: string // meta description
  url?: string         // "https://benettcar.hu"
  locale?: string      // "hu"
}

interface Navigation {
  primary: NavItem[]
  footer?: NavItem[]
}

interface NavItem {
  label: string        // "Rólunk"
  href: string         // "#about"
  children?: NavItem[]
}

interface Page {
  slug: string         // "home"
  title?: string       // "Főoldal"
  meta?: PageMeta
  sections: Section[]  // a page összes szekciója
}

interface Section {
  id: string           // "bc-hero"
  type: string         // "hero"
  data: unknown        // section-specifikus payload
}
```

### Hogyan lesz ebből JSON?

Egy konkrét (jövőbeli) példa Benett Car-ra:

```json
{
  "site": {
    "name": "Benett Car",
    "locale": "hu",
    "url": "https://benettcar.hu"
  },
  "navigation": {
    "primary": [
      { "label": "Szolgáltatások", "href": "#services" },
      { "label": "Rólunk", "href": "#about" },
      { "label": "Kapcsolat", "href": "#contact" }
    ]
  },
  "pages": [
    {
      "slug": "home",
      "title": "Benett Car",
      "sections": [
        {
          "id": "bc-hero",
          "type": "hero",
          "data": {
            "heading": "Prémium autóápolás",
            "subheading": "Professzionális szolgáltatások",
            "image": { "src": "/wp-content/uploads/hero.jpg", "alt": "..." }
          }
        },
        {
          "id": "bc-services",
          "type": "services",
          "data": {
            "items": [
              { "title": "Külső ápolás", "description": "..." },
              { "title": "Belső tisztítás", "description": "..." }
            ]
          }
        }
      ]
    }
  ]
}
```

### A formátum lényege

A `Section.data` mező típusa `unknown` — **szándékosan**. Minden section típusnak más a
payload-ja (hero-nak van képe és címe, services-nek van items listája, contact-nak van
email és cím). A React oldalon a `type` mező alapján a megfelelő komponens rendereli:

```
type: "hero"      → <HeroSection data={section.data} />
type: "services"  → <ServicesSection data={section.data} />
type: "contact"   → <ContactSection data={section.data} />
```

Ez a pattern lehetővé teszi, hogy **bármely kliensnél új section típust** lehessen
bevezetni anélkül, hogy az API-t vagy az engine-t módosítani kellene.

---

## Kliens config és overlay rendszer

### A probléma

A plugin forráskód (`sp-infra`) **reusable** — ugyanaz a kód fut minden kliensnek.
De minden kliens más:

| Kliens | Slug | Origins | Sections |
|---|---|---|---|
| Benett Car | `benettcar` | `localhost:5174` | 10 bc-* section |
| Jövőbeli kliens X | `clientx` | `localhost:5175` | 8 cx-* section |

### A megoldás: overlay

A kliens-specifikus config egy **külön repo-ban** él (`sp-benettcar`), és a plugin
futásidőben tölti be:

```php
// spektra-api.php
$spektra_config_path = getenv('SPEKTRA_CLIENT_CONFIG')
    ?: WP_PLUGIN_DIR . '/spektra-config/config.php';
```

A betöltés két stratégiát támogat:

| Stratégia | Hogyan | Mikor |
|---|---|---|
| **ENV var** | `SPEKTRA_CLIENT_CONFIG=/path/to/config.php` | Production, Docker |
| **Symlink fallback** | `plugins/spektra-config/` Junction → kliens repo | Lokális dev (WAMP) |

### A config tartalma

```php
// sp-benettcar/infra/config.php
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

A plugin ezt a tömböt a `SPEKTRA_CLIENT_CONFIG` PHP konstansba menti. Bármely osztály
hozzáfér: `SPEKTRA_CLIENT_CONFIG['client_slug']`.

### Az ACF mezők ugyanígy overlay-ből jönnek

```
spektra-config/
├── config.php            ← kliens beállítások
└── acf/
    └── field-groups.php  ← ACF mező definíciók (10 group, 51 field)
```

A `field-groups.php` saját `acf/init` hook-ban regisztrálja a field group-okat.
Így minden kliens saját section struktúrát definiálhat —  a plugin kódja változatlan.

---

## CORS — cross-origin hozzáférés

### A probléma

A frontend (`localhost:5174`) és a backend (`benettcar.local:80`) **különböző originen**
fut. A böngésző biztonsági modellje (Same-Origin Policy) alapértelmezetten **blokkolja**
a cross-origin kéréseket:

```
localhost:5174  →  GET http://benettcar.local/wp-json/spektra/v1/site
                   ❌ Blocked by CORS policy
```

### Mi az a CORS?

CORS (Cross-Origin Resource Sharing) egy HTTP header-alapú mechanizmus, amivel a
szerver **engedélyezi** a cross-origin kéréseket:

```
Böngésző → szervernek: "Hey, a localhost:5174-ről jövök."
Szerver → böngészőnek: "OK, az mehet." (Access-Control-Allow-Origin: http://localhost:5174)
Böngésző: "Rendben, átengedem a választ a JavaScript-nek."
```

### Hogyan működik ez concretely?

Két típusú kérés van:

#### 1. Simple request (egyszerű GET)

```
GET /wp-json/spektra/v1/site
Origin: http://localhost:5174

← 200 OK
← Access-Control-Allow-Origin: http://localhost:5174
```

A böngésző elküldi a kérést, a szerver a válasz header-ben jelzi, hogy az origin
engedélyezett. A böngésző átereszti.

#### 2. Preflight (OPTIONS kérés — előzetes egyeztetés)

Ha a kérésnek van speciális header-je (pl. `Authorization`, `Content-Type: application/json`),
a böngésző **előbb egy OPTIONS kérést** küld:

```
OPTIONS /wp-json/spektra/v1/site
Origin: http://localhost:5174
Access-Control-Request-Method: GET
Access-Control-Request-Headers: Authorization

← 204 No Content
← Access-Control-Allow-Origin: http://localhost:5174
← Access-Control-Allow-Methods: GET, OPTIONS
← Access-Control-Allow-Headers: Authorization, Content-Type
```

Ha az OPTIONS válasz OK → a böngésző elküldi a valódi GET-et.

### Honnan tudja a Spektra, ki az engedélyezett?

A kliens config-ból:

```php
'allowed_origins' => ['http://localhost:5174'],
```

A CORS class (`class-cors.php`) ezt olvassa ki, és csak a felsorolt origin-ekre küld
`Allow-Origin` headert. Ismeretlen origin → nincs header → böngésző blokkolja.

**Miért nem `Access-Control-Allow-Origin: *`?**

A `*` mindent átereszt — bármely weboldal JavaScript-je lekérheti az adatait.
Ez biztonsági kockázat (OWASP: Misconfigured CORS). A Spektra explicit listát használ.

---

## A kérés útja — teljes kép

Mit történik, amikor a React app betölti az oldalt:

```
1. Felhasználó megnyitja a localhost:5174-et
   │
2. React app mount → adapter.load() → fetch("/wp-json/spektra/v1/site")
   │
3. Böngésző: "Ez cross-origin kérés (5174 → :80). Van CORS engedély?"
   │
4. [Ha preflight kell] OPTIONS → CORS class → 204 + headerek
   │
5. GET /wp-json/spektra/v1/site
   │
   ├── Apache (WAMP) fogadja → WordPress betölti
   │   ├── wp-load.php → plugins betöltése
   │   │   ├── spektra-api.php → konstansok, autoload, config
   │   │   │   ├── config.php betöltés (Junction → sp-benettcar/infra/)
   │   │   │   └── ACF field-groups.php betöltés
   │   │   └── add_action('rest_api_init', ...)
   │   │
   │   └── rest_api_init → hook lefut
   │       ├── Rest_Controller::register_routes() → /spektra/v1/site
   │       └── CORS::register_hooks() → rest_pre_serve_request filter
   │
   ├── WP REST router: "Van /spektra/v1/site route." → handle_request()
   │   ├── preview param kiolvasása
   │   ├── Response_Builder::build() → ACF adatok összegyűjtése → SiteData JSON
   │   └── return WP_REST_Response (200, JSON)
   │
   └── CORS filter lefut: "Origin OK?" → Access-Control-Allow-Origin header
   │
6. Böngésző: "CORS header megvan." → válasz átadva a JavaScript-nek
   │
7. React: JSON → useState → render → felhasználó látja az oldalt
```

---

## WordPress REST API — miért erre építünk

### Mi az a WP REST API?

A WordPress 4.7 óta beépített REST API-val rendelkezik. Minden WordPress telepítés
az alábbi URL-en felsorolja az elérhető endpoint-okat:

```
GET /wp-json/
```

A natív API endpoint-okat kínál postokhoz, oldalakhoz, felhasználókhoz, stb.
A Spektra **nem használja ezeket közvetlenül**, hanem saját endpoint-ot regisztrál.

### Miért nem a natív WP REST endpoint-ok?

| WP natív endpoint | Probléma |
|---|---|
| `GET /wp-json/wp/v2/posts` | A formátum WP-specifikus, nem `SiteData` shape |
| `GET /wp-json/wp/v2/pages` | Nem tartalmaz ACF mezőket (külön plugin kellene) |
| `GET /wp-json/wp/v2/media` | Más struktúra, mint amit a frontend vár |

A natív endpoint-ok **általános célú** CMS API-t biztosítanak. A Spektra-nak viszont
**egyetlen, testreszabott** válasz kell, ami 1:1 megegyezik a frontend `SiteData`
típusával.

### A WordPress infrastruktúrát viszont használjuk

A WordPress REST framework-jét **kihasználjuk**:

| WP szolgáltatás | Spektra használat |
|---|---|
| `register_rest_route()` | Route regisztrálás a megfelelő namespace-szel |
| `WP_REST_Request` | Kérés paraméterek biztonságos kiolvasása |
| `WP_REST_Response` | Válasz küldése helyes HTTP státusszal és headerekkel |
| `rest_api_init` hook | Megfelelő életciklusban fut a regisztráció |
| `rest_pre_serve_request` filter | CORS headerek beillesztése a válaszba |
| `permission_callback` | Hozzáférés-kontroll (jelenleg `__return_true` = publikus) |

Ez nem „újrafeltaláljuk a kereket" — a WP REST framework robusztus, karbantartott,
és a Spektra pontosan azokat a részeit használja, amikre szüksége van.

---

## Mi különbözteti meg a Spektra API-t

### 1. Egy plugin → egy endpoint → egy kliens

A legtöbb WP REST plugin (WPGraphQL, ACF to REST API, Headless WP kiegészítők)
**általános célú**: minden post type-ot, taxonomiát, mezőt kiexportálnak.

A Spektra API **szándékosan szűk**: egyetlen endpoint, egyetlen válaszformátum,
egyetlen kliens. Ez nem korlát, hanem **tervezési döntés**.

| Általános REST plugin | Spektra API |
|---|---|
| 20+ endpoint | 1 endpoint |
| Generikus JSON | SiteData shape |
| Kliens-oldali transzformáció kell | Nincs transzformáció |
| Bármilyen frontend | Pontosan egy frontend |
| Sok query a DB felé | Egy összegyűjtött válasz |

### 2. A „szerződés" mindkét oldalon definiált

A legtöbb headless CMS rendszerben a frontend **alkalmazkodik** a backend válaszához.
A Spektra-ban fordítva is igaz: a `SiteData` TypeScript interfész (`sp-engine`) határozza
meg, mit vár a frontend, és a PHP Response_Builder **ezt a shape-et reprodukálja**.

```
sp-engine/src/types/SiteData.ts   ← szerződés (TypeScript)
     ↕ megegyezik
class-response-builder.php        ← szerződés megvalósítás (PHP)
```

Ha valaki módosítja a TypeScript interfészt, a PHP oldalt is frissíteni kell.
Ha valaki módosítja a PHP kimenetet, a TypeScript típusnak is követnie kell.
Ez **kétirányú szerződés**.

### 3. Config overlay — plugin ≠ kliens

A plugin forráskód (`sp-infra`) és a kliens config (`sp-benettcar`) **különböző repókban**
él. Egy plugin telepítés + egy overlay csere = **új kliens kész**.

Új kliens beüzemelése (elméletben):

```bash
# 1. Plugin már telepítve (sp-infra)
# 2. Kliens config overlay
mkdir sp-clients/sp-newclient/infra
# config.php + acf/field-groups.php → kliens-specifikus tartalom
# 3. Junction beállítás
# spektra-config → sp-newclient/infra
# 4. Kész — /spektra/v1/site az új kliens adatait adja
```

### 4. Preview támogatás

A `?preview=true` paraméter lehetővé teszi, hogy a WordPress admin felületen a
szerkesztő **real-time előnézetet** kapjon a módosításairól a React frontend-en,
mielőtt publikálna.

---

## Aktuális állapot

> ⚠️ **Ez a szekció a fejlesztés aktuális fázisát dokumentálja. Véglegesítés után
> törlendő vagy archíválandó.**

### Mi kész?

| Komponens | Státusz | Commit | Megjegyzés |
|---|---|---|---|
| `spektra-api.php` | ✅ Kész | `9b0321f` (P5.1) | Konstansok, autoload, config, hook reg. |
| Plugin aktiválás | ✅ Kész | `49d5a96` (P4.3) | WP runtime-ban aktív |
| Client overlay | ✅ Kész | `1f9db68` (P4.4) | Junction → sp-benettcar/infra |
| ACF field groups | ✅ Kész | `43c4456` (P4.5) | 10 group, 51 field |
| `class-rest-controller.php` | ✅ Kész | `0f7a129` (P5.2) | Schema, validate, sanitize, headers |
| `class-cors.php` | ✅ Kész | `26a3f21` (P5.3) | Whitelist, preflight 204, Vary, header_remove |
| `class-response-builder.php` | 🔶 Placeholder | — | Phase 7-ben lesz valódi assembly |
| Config loading | ✅ Verified | — (P5.4) | 5 config kulcs, 10 ACF group, 51 field — runtime-log #6 |
| Smoke test | ✅ 18/18 PASS | — (P5.5) | REST + CORS + preview + preflight — runtime-log #7 |

### Mi jön?

| Fázis | Lépés | Mit csinál |
|---|---|---|
| Phase 6 | P6.1 | Reusable ACF helpers (`spektra_get_field`, `spektra_normalize_media`) |
| Phase 6 | P6.2–6.4 | ACF field registration verify, admin verify, test data |
| Phase 7 | P7.1 | Response_Builder — osztályváz, section assembly interfész |
| Phase 7 | P7.2 | Site meta + Navigation assembly |
| Phase 7 | P7.3 | Section assembly — ACF → SiteData sections[] |
| Phase 7 | P7.4 | Media normalization — ACF image → `{ src, alt, width, height }` |
| Phase 7 | P7.5 | Teljes SiteData endpoint teszt |

### Jelenlegi endpoint válasz

```json
{
  "site": [],
  "navigation": [],
  "pages": []
}
```

Ez **szándékos placeholder** — a HTTP layer (route, CORS, headerek) kész, a tartalom Phase 7-ben jön.

---

## Összefoglaló

| Kérdés | Válasz |
|---|---|
| Mi az a Spektra API? | WP REST plugin, ami SiteData JSON-t szolgáltat |
| Miért kell? | A React frontend és a WordPress backend között nincs más híd |
| Hány endpoint van? | Egy: `GET /wp-json/spektra/v1/site` |
| Mi a válasz formátuma? | `SiteData` — TypeScript interfész alapján |
| Honnan jön a tartalom? | ACF mezőkből + kliens config-ból |
| Hogyan tud egy másik kliens is használni? | Overlay csere: `spektra-config/` → másik repo |
| Mi a CORS? | HTTP header mechanizmus, ami megengedi a cross-origin kéréseket |
| Hol van a szerződés? | `sp-engine/src/types/SiteData.ts` (TS) ↔ `Response_Builder` (PHP) |
| Milyen státuszban van? | HTTP layer kész (Phase 5 complete), Response Builder placeholder (Phase 7) |
