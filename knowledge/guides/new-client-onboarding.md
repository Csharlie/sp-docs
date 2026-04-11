# New Client Onboarding Guide

> Standalone walkthrough — új Spektra kliens projekt bootstrapja a preset-alapú rendszerrel.

---

## Előfeltételek

| Elem | Leírás |
|------|--------|
| **Node.js** | >= 18 LTS |
| **pnpm** | >= 9 (workspace protocol support) |
| **PHP** | 8.4.x (WAMP — 8.5+ breaks WP-CLI phar) |
| **WordPress** | Lokális WP futtatókörnyezet (WAMP/LocalWP) |
| **ACF Pro** | Aktiválva a WP adminban |
| **Repók klónozva** | `sp-platform`, `sp-infra`, `sp-clients`, `sp-docs` egy közös `spektra/` gyökér alatt |

Workspace elvárt struktúra:

```
spektra/
├── sp-platform/          # Közös engine + layouts + presets
├── sp-infra/             # WP plugin, scripts, seed pipeline
├── sp-clients/
│   └── sp-{slug}/        # <-- EZT HOZZUK LÉTRE
├── sp-docs/              # Knowledge base
└── .local/
    └── wp-runtimes/
        └── {slug}/       # WP core + plugins (gitignored)
```

---

## 1. Preset kiválasztás

A preset határozza meg, milyen termékvonalhoz tartozik a kliens.

| Preset | Jellemző | Platform package | Infra preset |
|--------|----------|------------------|--------------|
| **baseline** | Minimális, custom layout | `@spektra/preset-baseline` | `sp-infra/presets/baseline/` |
| **landingline** | Landing page (hero, CTA, sections) | `@spektra/preset-landingline` | `sp-infra/presets/landingline/` |
| **shopline** | E-commerce / termékoldal | `@spektra/preset-shopline` | `sp-infra/presets/shopline/` |

A `sp-benettcar` referencia implementáció a **landingline** vonalba esik (hero, gallery, services, about, contact sections).

---

## 2. Kliens projekt létrehozása

### 2.1 Mappa struktúra

```powershell
$Slug = "newclient"   # <-- kliens slug (kebab-case)
$ClientDir = "D:\Projects\spektra\sp-clients\sp-$Slug"

New-Item -ItemType Directory -Path $ClientDir -Force
New-Item -ItemType Directory -Path "$ClientDir\src" -Force
New-Item -ItemType Directory -Path "$ClientDir\infra\acf\sections" -Force
New-Item -ItemType Directory -Path "$ClientDir\infra\env" -Force
New-Item -ItemType Directory -Path "$ClientDir\infra\seed" -Force
New-Item -ItemType Directory -Path "$ClientDir\infra\docs" -Force
```

### 2.2 package.json

```json
{
  "name": "@spektra/sp-newclient",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint src/",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@spektra/layouts": "workspace:*",
    "@spektra/engine": "workspace:*"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "vite": "^6.0.0",
    "vitest": "^3.0.0"
  }
}
```

> **Fontos:** A `workspace:*` protocol a pnpm workspace linkelést használja az sp-platform-ből. A `pnpm-workspace.yaml`-ban a kliens projektnek benne kell lennie.

---

## 3. Infra overlay konfigurálás

### 3.1 config.php

Az overlay belépési pontja. A `spektra-api.php` plugin innen tölti a kliens identitást, CORS beállításokat, és section rendet.

```php
<?php
// infra/config.php

return [
    // ── Client identity ──────────────────────────────────────
    'client_slug'  => 'newclient',
    'client_name'  => 'New Client Kft.',

    // ── CORS ──────────────────────────────────────────────────
    'allowed_origins' => [
        'http://localhost:5174',  // Vite dev server
    ],

    // ── Site defaults ─────────────────────────────────────────
    'site_defaults' => [
        'lang'  => 'hu',
        'title' => 'New Client',
    ],

    // ── Navigation ────────────────────────────────────────────
    'navigation' => [
        'primary' => [
            ['label' => 'Főoldal', 'href' => '/'],
            // ... további elemek
        ],
        'footer'  => [
            ['label' => 'Impresszum', 'href' => '/impresszum'],
            // ... további elemek
        ],
    ],

    // ── Sections ──────────────────────────────────────────────
    // A sorrend = site.ts pages[home].sections[] renderelési sorrend!
    'sections' => [
        'nc-hero',
        'nc-about',
        'nc-contact',
    ],
];
```

> **Szabály:** A `sections` lista sorrendje meghatározza a frontend renderelési sorrendet. A prefix (`nc-`) a kliens slug rövidítése — konvenció, nem kényszer.

### 3.2 ACF Field Groups

`infra/acf/field-groups.php` — az ACF registry. Ez regisztrálja a section-ök Custom Field group-jait WP-ben.

```php
<?php
// infra/acf/field-groups.php
// Loaded by spektra-api.php on acf/init hook

$sectionsDir = __DIR__ . '/sections';

foreach (glob("$sectionsDir/*.php") as $file) {
    require_once $file;
}
```

### 3.3 Section definíciók

Minden section egy `infra/acf/sections/{prefix}-{name}.php` fájl. Referencia: `sp-benettcar/infra/acf/sections/`.

Struktúra:

```php
<?php
// infra/acf/sections/nc-hero.php

if (function_exists('acf_add_local_field_group')) {
    acf_add_local_field_group([
        'key'    => 'group_nc_hero',
        'title'  => 'NC Hero',
        'fields' => [
            [
                'key'   => 'field_nc_hero_title',
                'label' => 'Title',
                'name'  => 'title',
                'type'  => 'text',
            ],
            [
                'key'   => 'field_nc_hero_image',
                'label' => 'Background Image',
                'name'  => 'backgroundImage',
                'type'  => 'image',
                'return_format' => 'url',
            ],
        ],
        'location' => [
            [['param' => 'options_page', 'operator' => '==', 'value' => 'spektra-config']],
        ],
    ]);
}
```

### 3.4 Builders

`infra/acf/builders.php` — section builder dispatch. A generic registry a `config.php` sections listáját használja, nincs hardcoded switch.

### 3.5 Environment

`infra/env/.env.example` másolása → `infra/env/.env`:

```bash
# WordPress local site URL
SPEKTRA_WP_URL=http://newclient.local

# Abszolút útvonal az infra/config.php-hez
SPEKTRA_CLIENT_CONFIG=D:\Projects\spektra\sp-clients\sp-newclient\infra\config.php

# Vite dev server origin (CORS-nak meg kell egyeznie config.php-vel)
VITE_DEV_ORIGIN=http://localhost:5174

# WAMP paths
WAMP_WWW=D:\Local\wamp\www
WAMP_VHOST_DIR=D:\Local\wamp\bin\apache\apache2.4.65\conf\extra

# Debug
SPEKTRA_WP_DEBUG=true
SPEKTRA_DEBUG=true
```

> **Fontos:** `.env` gitignored, `.env.example` verziókezelt.

### 3.6 Content Model docs

`infra/docs/content-model.md` — Single source of truth a field nevekhez. Minden field amit az ACF definícióban használsz, ide dokumentáld.

---

## 4. WordPress runtime beállítás

### 4.1 Runtime mappa

```powershell
$Slug = "newclient"
$RuntimeDir = "D:\Projects\spektra\.local\wp-runtimes\$Slug"
New-Item -ItemType Directory -Path $RuntimeDir -Force
```

IDE: WordPress core telepítés (manuálisan vagy WP-CLI-vel).

### 4.2 Plugin symlinkelés

```powershell
# Spektra API plugin (közös, sp-infra-ból):
.\sp-infra\scripts\link-plugin.ps1 -Client newclient

# Kliens overlay (sp-clients-ből):
.\sp-infra\scripts\link-overlay.ps1 -Client newclient
```

Eredmény:
```
.local/wp-runtimes/newclient/wp-content/plugins/
├── spektra-api/      → junction → sp-infra/plugin/spektra-api/
└── spektra-config/   → junction → sp-clients/sp-newclient/infra/
```

### 4.3 ACF aktiválás

1. WP admin → Plugins → ACF Pro aktiválás
2. WP admin → Plugins → Spektra API aktiválás
3. Az ACF field group-ok automatikusan regisztrálódnak az `acf/init` hook-on

### 4.4 WAMP vhost (opcionális)

A `SPEKTRA_WP_URL`-nek megfelelő vhost létrehozása a WAMP-ban (`newclient.local`).

---

## 5. Seed pipeline

Miután a WP runtime felállt és az ACF mezők regisztrálva vannak:

```powershell
# Teljes pipeline (export → import → dump → verify → smoke):
.\sp-infra\seed\seed-pipeline.ps1 -Client newclient

# Vagy lépésenként:
.\sp-infra\seed\seed-pipeline.ps1 -Client newclient -DryRun     # Száraz futtatás
.\sp-infra\seed\seed-pipeline.ps1 -Client newclient -Verbose     # Részletes log
.\sp-infra\seed\seed-pipeline.ps1 -Client newclient -SkipExport  # Meglévő seed.json használata
```

---

## 6. Frontend indítás

```powershell
cd sp-clients\sp-newclient
pnpm install
pnpm dev          # Vite dev server → http://localhost:5174
```

---

## 7. Ellenőrzés

| Lépés | Parancs / művelet | Elvárt eredmény |
|-------|-------------------|-----------------|
| Build | `pnpm build` | Hiba nélkül lefut |
| Test | `pnpm test` | Összes teszt zöld |
| REST endpoint | `curl http://newclient.local/wp-json/spektra/v1/site` | JSON response, sections adattal |
| CORS | Vite → WP fetch | Nincs CORS error a böngészőben |
| Image fields | REST response-ban image URL-ek | Nem `false`, hanem érvényes URL |

---

## Referencia: sp-benettcar struktúra

A `sp-clients/sp-benettcar` a jelenleg egyetlen teljes referencia implementáció:

```
sp-benettcar/
├── package.json              # @spektra/sp-benettcar, scripts: dev/build/test/seed:*
├── vite.config.ts            # Port 5174
├── tsconfig.json
├── src/
│   ├── App.tsx               # Router + LayoutShell import
│   ├── site.ts               # SiteData definíció (pages, sections, nav)
│   └── shell/
│       ├── Header.tsx        # LayoutShellProps → kliens header
│       └── Footer.tsx        # LayoutShellProps → kliens footer
└── infra/
    ├── config.php            # Client identity, CORS, sections
    ├── acf/
    │   ├── field-groups.php  # ACF registry (acf/init hook)
    │   ├── builders.php      # Section builder dispatch
    │   └── sections/         # 10 section (bc-hero, bc-brand, ...)
    ├── env/
    │   ├── .env.example      # Template
    │   └── .env              # Gitignored local paths
    ├── seed/
    │   ├── export-seed.ts    # site.ts → seed.json exporter
    │   ├── mapping.ts        # Field mapping definíciók
    │   └── register-asset-hooks.mjs
    └── docs/
        ├── README.md         # Overlay guide
        └── content-model.md  # Field contract (P3.2)
```

---

## Gyakori hibák

| Hiba | Ok | Megoldás |
|------|----|----------|
| `TS2307: Cannot find module '@spektra/layouts'` | pnpm workspace link hiányzik | `pnpm install` az sp-platform gyökérből |
| CORS error böngészőben | `allowed_origins` nem egyezik | `config.php` + `.env` `VITE_DEV_ORIGIN` összehangolás |
| ACF field group-ok nem jelennek meg | Plugin nem aktív / overlay nem linkelt | `link-overlay.ps1 -Client slug` + WP admin plugin aktiválás |
| `seed-pipeline.ps1` PHP error | Rossz PHP verzió | Ellenőrizd: `php -v` → 8.4.x kell |
| `Overlay source not found` | Rossz slug vagy hiányzó infra mappa | `sp-clients/sp-{slug}/infra/` léteznie kell |

---

## Kapcsolódó dokumentáció

- `sp-docs/spektra-api-guide.md` — API plugin felépítés
- `sp-docs/cors-guide.md` — CORS konfiguráció részletek
- `sp-docs/smoke-test-guide.md` — Smoke test részletek
- `sp-docs/knowledge/concepts/seed-pipeline.md` — Seed pipeline koncepció
- `sp-docs/knowledge/phases/phase-11.3.md` — Preset struktúra háttér
- `sp-docs/knowledge/phases/phase-11.4.md` — Script parameterization
