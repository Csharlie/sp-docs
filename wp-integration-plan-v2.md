## WordPress integráció — Javított terv v2

---

### 0. Platform Change Policy — WP Integration Phase

#### Context

A WordPress integráció során a Spektra platform még építés alatt áll.
Ezért platform módosítások megengedettek, ha azok javítják az architektúrát.

#### Allowed Platform Changes

**Adapter contract módosítás**
Ha a WordPress integráció igényli:
- loading state támogatás
- error state támogatás
- retry logika
- preview mód támogatás

A `createWordPressAdapter` miatt ezek platformszintűvé válhatnak.

**Media contract módosítás**
WordPress image adatok: sizes, alt, mime, responsive variánsok.
Ha a jelenlegi `Media` shape nem elég — bővíthető, finomítható, normalizálható.
Ez platform módosításnak számít.

**Validation szabályok módosítása**
WP esetén: hiányzó mezők, opcionális mezők, partial content.
Ha szükséges: `validateSiteData` módosítható, validation lazítható, fallback policy bővíthető.

**Section payload rugalmasság**
Ha WP miatt szükséges: optional mezők, partial payload, dynamic mezők.
Platform szinten módosítható.

#### Not Allowed Platform Changes

**WordPress-specifikus logika a platformban**
Nem kerülhet a platformba: ACF logika, WP endpoint logika, WP image field logika, WP meta logika.
Ezek az adapter / mapper rétegben maradnak.

**Client-specifikus mezők a platformban**
Nem kerülhet a platformba: `bc-hero`, `bc-gallery`, `bc-contact`, stb.
A platform section-agnosztikus marad.

#### Decision Rule

Ha a WordPress integráció során:
- workaround kell a kliensben → platform módosítás **megfontolandó**
- mapping túl bonyolult → platform módosítás **megfontolandó**
- validation túl szigorú → platform módosítás **megfontolandó**

#### Goal

- WordPress integráció működjön
- Platform stabilabb legyen
- Új kliensekhez ne kelljen platform módosítás

---

### 1. Mappaszerkezet (javított)

`sp-infra` **mappa** (nem repo), a `D:\Projects\spektra\` gyökérben:

```
D:\Projects\spektra\
├── sp-platform\                         # NEM VÁLTOZIK
├── sp-clients\sp-benettcar\             # MÓDOSUL (adapter switch + mapper)
└── sp-infra\                            # ÚJ MAPPA (nem repo)
    └── sp-benettcar\
        ├── docker-compose.yml
        ├── .env.example
        ├── .env                         # gitignored
        ├── wp-config-local.php          # sablon
        ├── scripts\
        │   ├── export-seed.ts           # Node: site.ts → seed.json
        │   └── seed.json                # generált, gitignored
        └── wp\
            └── plugins\
                └── spektra-api\
                    ├── spektra-api.php
                    ├── includes\
                    │   ├── class-rest-controller.php
                    │   ├── class-response-builder.php
                    │   └── class-cors.php
                    └── clients\
                        └── benettcar\
                            ├── config.php
                            └── acf-fields.php
```

**Megjegyzés a dokumentációról:** a terv jelenleg `sp-docs/` alatt él. Eldöntendő, hogy ez a mappa marad-e külön,
vagy a későbbiekben a **platform-agnosztikus** részek `sp-platform/docs/` alá, a **kliens-specifikus** részek pedig
`sp-clients/sp-benettcar/docs/` alá kerülnek.

---

### 2. WP response shape (bővített)

`GET /wp-json/spektra/v1/site` — teljes SiteData-kompatibilis válasz:

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
      { "label": "Adatvédelem", "href": "/adatvedelem" },
      { "label": "ÁSZF", "href": "/aszf" }
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
        "ogImage": {
          "src": "https://benettcar.local/wp-content/uploads/og.jpg",
          "alt": "Benett Car",
          "width": 1200,
          "height": 630
        }
      },
      "sections": [
        {
          "id": "hero",
          "type": "bc-hero",
          "data": {
            "title": "Benett Car",
            "subtitle": "...",
            "description": "...",
            "primaryCTA": { "text": "Ajánlataink", "href": "#gallery" },
            "secondaryCTA": { "text": "Kapcsolat", "href": "#kapcsolat" },
            "backgroundImage": {
              "src": "https://benettcar.local/wp-content/uploads/hero-bg.jpg",
              "alt": "Benett Car showroom",
              "width": 1920,
              "height": 1080,
              "variants": [
                {
                  "name": "thumbnail",
                  "source": { "src": ".../hero-bg-300x200.jpg", "width": 300, "height": 200 }
                }
              ]
            }
          }
        }
      ]
    }
  ]
}
```

**Szabály**: minden kép mező (backgroundImage, logo, image, ogImage) **Media shape** — `{ src, alt, width?, height?, variants? }`. A `class-response-builder.php` felelős az ACF image mezőt → Media formára konvertálni. A `variants[].source` mező `MediaSource` kompatibilis (`url`, `width?`, `height?`, `format?`). Fontos: a `Media` gyökérszinten `src`-t használ, a `MediaSource`-ban viszont `url`-t.

---

### 3. Section data konzisztencia

A WP `sections[].data` **pontosan** a kliens section regisztráció által várt shape-et adja vissza. A 10 benettcar szekció-típus:

| type | Kulcs data mezők |
|---|---|
| `bc-hero` | title, subtitle, description, primaryCTA, secondaryCTA, backgroundImage (Media) |
| `bc-brand` | title, description, brands[] (name + logo: Media) |
| `bc-gallery` | title, subtitle, showCategories, images[] (Media + category?) |
| `bc-services` | title, subtitle, services[] (icon, title, description) |
| `bc-service` | title, subtitle, description, services[], brands[], contact |
| `bc-about` | title, subtitle, content[], image (Media), imagePosition, stats[], cta, colorScheme |
| `bc-team` | title, subtitle, description, members[] (name, role, image: Media, bio?) |
| `bc-assistance` | title, subtitle, description, requestLabel, requestHref, serviceArea |
| `bc-contact` | title, subtitle, description, contactInfo, colorScheme |
| `bc-map` | title, query, height |

Az ACF field group-ok **ezekre a mezőkre épülnek** — ha a WP más shape-et ad, a szekció-komponens törni fog. A content model spec (3. lépés) rögzíti az 1:1 mapping-et.

---

### 4. CORS + preflight (bővített)

`class-cors.php`:

```php
class Spektra_CORS {
    private array $allowed_origins;

    public function __construct(array $origins) {
        $this->allowed_origins = $origins;
    }

    public function register(): void {
        add_action('rest_api_init', [$this, 'handle_cors']);
        add_filter('rest_pre_serve_request', [$this, 'add_cors_headers']);
    }

    public function handle_cors(): void {
        // OPTIONS preflight kezelés
        if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
            $origin = $_SERVER['HTTP_ORIGIN'] ?? '';
            if (in_array($origin, $this->allowed_origins, true)) {
                header("Access-Control-Allow-Origin: $origin");
                header('Access-Control-Allow-Methods: GET, OPTIONS');
                header('Access-Control-Allow-Headers: Authorization, Content-Type');
                header('Access-Control-Allow-Credentials: true');
                header('Access-Control-Max-Age: 86400');
                status_header(204);
                exit;
            }
        }
    }

    public function add_cors_headers($value) {
        $origin = get_http_origin();
        if (in_array($origin, $this->allowed_origins, true)) {
            header("Access-Control-Allow-Origin: $origin");
            header('Access-Control-Allow-Credentials: true');
        }
        return $value;
    }
}
```

**Konfigurálva** `clients/benettcar/config.php`-ban:
- Dev: `['http://localhost:5174', 'http://localhost:5175']`
- Prod: `['https://benettcar.hu']`

---

### 5. Auth stratégia

**Döntés**: az endpoint **alapból publikus** (read-only site data). Token **opcionális**, draft/preview content-hez.

| Mód | Endpoint | Auth |
|---|---|---|
| Publikus oldal | `GET /wp-json/spektra/v1/site` | nincs |
| Draft preview | `GET /wp-json/spektra/v1/site?preview=true` | `Authorization: Bearer <token>` |

**Szabály**: publikus endpoint csak publish tartalmat ad vissza, preview módban draft/private tartalom is jöhet.

A `createWordPressAdapter` `auth` paramétere csak preview módban kap token-t. Az `.env`-ben:

```
VITE_DATA_SOURCE=wordpress
VITE_WP_API_BASE=http://benettcar.local
VITE_WP_AUTH_TOKEN=          # üres = publikus, kitöltve = preview mód
```

---

### 6. Seed stratégia (javított)

**Két lépcsős**: Node export → WP-CLI import. A PHP **nem parse-olja** a site.ts-t.

```
sp-infra/sp-benettcar/scripts/
├── export-seed.ts    # 1. lépés: site.ts → seed.json
├── seed.json         # generált (gitignored)
└── package.json      # tsx, minimál
```

**export-seed.ts** (pszeudó):
```ts
import { siteData } from '../../../sp-clients/sp-benettcar/src/data/site'
// siteData → WP REST create post payloadok
// Kimenet: seed.json = { posts: [...], acf_values: [...], media: [...] }
```

**WP-CLI import** (konténerből vagy WAMP-ból):
```bash
wp spektra import --file=seed.json
```

A `spektra-api` plugin regisztrál egy WP-CLI parancsot, ami a `seed.json`-t olvassa be és létrehozza a WP tartalmat (pages, ACF mezők, media uploads).

---

### 7. Fallback viselkedés

**App.tsx** — explicit kezelés:

```tsx
const adapter =
  import.meta.env.VITE_DATA_SOURCE === 'wordpress'
    ? createWordPressAdapter({
        apiBase: import.meta.env.VITE_WP_API_BASE,
        mapResponse,
        auth: import.meta.env.VITE_WP_AUTH_TOKEN
          ? { token: import.meta.env.VITE_WP_AUTH_TOKEN }
          : undefined,
      })
    : createJsonAdapter({ data: siteData })
```

| Helyzet | Viselkedés |
|---|---|
| `VITE_DATA_SOURCE=json` | Statikus site.ts, WP nem kell |
| `VITE_DATA_SOURCE=wordpress`, WP elérhető | Éles WP adat |
| `VITE_DATA_SOURCE=wordpress`, WP nem elérhető | `SiteDataProvider` `loading` → `onError` → hiba UI |

**Nincs automatikus JSON fallback prod-ban** — ha WP-re van állítva és nem elérhető, az hiba, nem csendes fallback. Dev-ben az env switch elegendő.

---

### 8. Docker (teljes)

**docker-compose.yml**:

```yaml
services:
  wordpress:
    image: wordpress:6.7-php8.3-apache
    ports:
      - "${WP_PORT:-8080}:80"
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
      - ./wp/plugins/spektra-api:/var/www/html/wp-content/plugins/spektra-api
      - wp-uploads:/var/www/html/wp-content/uploads
    depends_on:
      mariadb:
        condition: service_healthy

  mariadb:
    image: mariadb:11.4
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
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
      - ./scripts:/scripts
    depends_on:
      - wordpress
    entrypoint: ["wp", "--allow-root"]
    profiles: ["cli"]

volumes:
  wp-data:
  db-data:
  wp-uploads:
```

**.env.example** (Docker-hez):
```
MYSQL_DATABASE=spektra_benettcar
MYSQL_USER=spektra
MYSQL_PASSWORD=changeme
MYSQL_ROOT_PASSWORD=rootchangeme
WP_PORT=8080
```

**Domain**: `localhost:8080` dev-ben. Ha `benettcar.local` kell → hosts fájl entry. Prod esetén reverse proxy (nginx/traefik) HTTPS-sel — de az a deploy fázis, nem most.

**WP-CLI seed futtatás**:
```bash
docker compose run --rm wpcli spektra import --file=/scripts/seed.json
```

**wp-uploads volume** biztosítja, hogy a feltöltött képek megmaradjanak konténer újraindítás/törlés után.

---

### 9. Végrehajtási sorrend (javított)

| # | Művelet | Hol | Előfeltétel |
|---|---|---|---|
| **1** | `sp-infra/sp-benettcar/` mappa + docker-compose + .env | `sp-infra/` | — |
| **2** | `spektra-api` plugin skeleton (PHP bootstrap + REST route + CORS) | `sp-infra/.../spektra-api/` | — |
| **3** | Content model spec (ACF mező ↔ site.ts kulcs, 10 szekció) | dokumentum | site.ts átnézés |
| **4** | ACF field group-ok PHP regisztrálása | `acf-fields.php` | #3 |
| **5** | Response builder (ACF → SiteData-kompatibilis JSON, Media shape) | `class-response-builder.php` | #4 |
| **6** | Docker up + WP install + plugin aktiválás + ACF Pro | Docker | #1, #2 |
| **7** | Seed: `export-seed.ts` (Node) + `wp spektra import` (WP-CLI) | `scripts/` + Docker | #5, #6 |
| **8** | `wp-mapper.ts` + `.env` + `App.tsx` adapter switch | `sp-clients/sp-benettcar/` | #5 |
| **9** | E2E dev validáció (WP admin → frontend, CORS, Media) | dev loop | #6, #8 |
| **10** | Stabilizáció + fallback UX + doku | mindenhol | #9 |

---

**sp-platform: nulla módosítás.** Jóváhagyod, vagy van még finomítás?
