# Spektra -- Runtime Setup Log

Kronologikus naplo a lokalis runtime konfiguracios lepesekrol.
Ezek a lepesek **nem verziokezett** fajlokat erintenek (WAMP config, hosts, WP install, symlink-ek, ACF plugin),
ezert kulon logoljuk oket ide.

**Remote:** https://github.com/Csharlie/sp-docs
**Scope:** Phase 4 (WordPress Runtime Assembly) es kesobb barmi, ami host-specifikus runtime setup

---

## Naplo

| # | Phase | Lepes | Datum | Statusz |
|---|-------|-------|-------|---------|
| 1 | P4.1 | WAMP vhost | 2026-04-05 | done |
| 2 | P4.2 | WordPress install | 2026-04-05 | done |
| 3 | P4.3 | Plugin symlink + activate | 2026-04-05 | done |
| 4 | P4.4 | Client overlay linking | 2026-04-05 | done |
| 5 | P4.5 | ACF install + verify | 2026-04-05 | done |
| 6 | P5.4 | Config loading verification | 2026-04-05 | done |
| 7 | P5.5 | Phase 5 full smoke test | 2026-04-05 | done |
| 8 | P6.2 | Client field registration verify | 2026-04-05 | done |
| 9 | P6.3 | Field definition integrity audit | 2026-04-05 | done |

---

## #1 -- WAMP vhost: benettcar.local (2026-04-05) -- P4.1

**Cel:** `http://benettcar.local` valaszoljon Apache-bol.

### Mi tortent

1. **Runtime konyvtar letrehozva** (PowerShell):
   ```powershell
   New-Item -ItemType Directory -Path "D:\Projects\spektra\.local\wp-runtimes\benettcar" -Force
   ```
   - Teszt `index.html` elhelyezve (VS Code create file)

2. **Apache vhost hozzaadva** (VS Code szerkesztes):
   - Fajl: `D:\Local\wamp\bin\apache\apache2.4.65\conf\extra\httpd-vhosts.conf`
   - Meglevo localhost blokk erintetlen, uj blokk utana:
   ```apache
   <VirtualHost *:80>
     ServerName benettcar.local
     DocumentRoot "D:/Projects/spektra/.local/wp-runtimes/benettcar"
     <Directory "D:/Projects/spektra/.local/wp-runtimes/benettcar/">
       Options FollowSymLinks
       AllowOverride All
       Require all granted
     </Directory>
   </VirtualHost>
   ```

3. **Hosts fajl bovitve** (elevated PowerShell / kezzel):
   ```powershell
   # Admin jogosultsag szukseges!
   Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`n127.0.0.1    benettcar.local"
   ```

4. **WAMP ujrainditva** (WAMP tray ikon → Restart All Services)

### Architekturalis dontesek

- **DocumentRoot NEM `wamp/www/` alatt** -- hanem `D:\Projects\spektra\.local\wp-runtimes/benettcar`
  - Runtime a Spektra projekt fan belul marad
  - `.local/` gitignored
  - Multi-client ready (kesobbi kliensek: `.local/wp-runtimes/<slug>/`)
- **`Require all granted`** a `Require local` helyett -- Vite dev server fetch, REST API, CORS mind mukodjon
- **`Options FollowSymLinks`** -- Indexes kihagyva (WP-nel felesleges directory listing)

### Smoke test

```
Invoke-WebRequest http://benettcar.local --> 200 OK
"benettcar.local works"
```

### Statusz

Kesz.

---

## #2 -- WordPress install (2026-04-05) -- P4.2

**Cel:** WordPress runtime a benettcar vhost-ban.

### Mi tortent

Minden parancs **PowerShell**-bol futott (VS Code terminal).

1. **WordPress 6.9.4 letoltve es kicsomagolva:**
   ```powershell
   # Letoltes
   $zipPath = "$env:TEMP\wordpress-latest.zip"
   Invoke-WebRequest -Uri "https://wordpress.org/latest.zip" -OutFile $zipPath -UseBasicParsing

   # Kicsomagolas temp-be
   $extractPath = "$env:TEMP\wp-extract"
   Expand-Archive -Path $zipPath -DestinationPath $extractPath -Force

   # Masolasa runtime konyvtarba (root-ba, nem alkonyvtarba)
   $src = "$env:TEMP\wp-extract\wordpress"
   $dst = "D:\Projects\spektra\.local\wp-runtimes\benettcar"
   Remove-Item "$dst\index.html" -Force
   Copy-Item -Path "$src\*" -Destination $dst -Recurse -Force

   # Temp takaritas
   Remove-Item $zipPath -Force
   Remove-Item $extractPath -Recurse -Force
   ```

2. **Adatbazis letrehozva** (PowerShell → WAMP MySQL):
   ```powershell
   & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root -e "
     CREATE DATABASE IF NOT EXISTS sp_benettcar
     CHARACTER SET utf8mb4
     COLLATE utf8mb4_unicode_ci;
   "
   ```
   - Nev: `sp_benettcar` (Spektra naming: `sp_{client_slug}`)

3. **wp-config.php letrehozva** (VS Code create file):
   - DB: `sp_benettcar`, root, ures jelszo, localhost
   - Debug: `WP_DEBUG=true`, `WP_DEBUG_LOG=true`, `WP_DEBUG_DISPLAY=false`
   - Salt generalas:
     ```powershell
     (Invoke-WebRequest -Uri "https://api.wordpress.org/secret-key/1.1/salt/" -UseBasicParsing).Content
     ```
   - Spektra config: **NEM** hozzaadva (Phase 5)

4. **WordPress telepites** (PowerShell → HTTP POST az installerhez):
   ```powershell
   $body = @{
     weblog_title    = 'Benett Car'
     user_name       = 'admin'
     admin_password  = 'admin'
     admin_password2 = 'admin'
     pw_weak         = '1'
     admin_email     = 'dev@benettcar.local'
     blog_public     = '0'
     language         = 'hu_HU'
   }
   Invoke-WebRequest -Uri "http://benettcar.local/wp-admin/install.php?step=2" `
     -Method POST -Body $body -UseBasicParsing
   ```
   - Site title: `Benett Car`
   - Admin: `admin` / `admin` (dev only)
   - Email: `dev@benettcar.local`
   - Search engine visibility: off

### Smoke test

```
http://benettcar.local        --> 200 OK
http://benettcar.local/wp-admin/ --> 200 OK
http://benettcar.local/wp-json/  --> 200 OK
```

### Guardrails betartva

- Nem telepitettem plugint
- Nem telepitettem themet
- Nem adtam hozza Spektra configot
- Nem hoztam letre symlinket
- Nem modositottam wp-content-et

### Statusz

Kesz.

---

## #3 -- Plugin symlink + activate (2026-04-05) -- P4.3

**Cel:** Spektra API plugin elerheto WP-ben Junction-on keresztul, aktivalva, nincs fatal error.

### Mi tortent

Minden parancs **PowerShell**-bol futott (VS Code terminal).

1. **link-plugin.ps1 frissitve** (sp-infra commit):
   - Scaffold lecserelve valos implementaciora
   - `New-Item -ItemType Junction` (nem SymbolicLink -- admin jog nem kell)
   - Idempotens: `Test-Path` guard

2. **Junction letrehozva** (PowerShell):
   ```powershell
   cd D:\Projects\spektra\sp-infra
   & .\scripts\link-plugin.ps1 -Client benettcar
   ```
   Eredmeny:
   ```
   .local\wp-runtimes\benettcar\wp-content\plugins\spektra-api
     --> D:\Projects\spektra\sp-infra\plugin\spektra-api
   ```

3. **Plugin aktivalas** (PowerShell --> WAMP MySQL):
   ```powershell
   echo 'UPDATE wp_options SET option_value=''a:1:{i:0;s:27:"spektra-api/spektra-api.php";}'' WHERE option_name=''active_plugins'';' | & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar
   ```
   Fontos: `s:27` -- a string hossz pontosan 27 karakter!

4. **Permalink struktura** (PowerShell --> WAMP MySQL):
   ```powershell
   & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar -e "UPDATE wp_options SET option_value='/%postname%/' WHERE option_name='permalink_structure'"
   & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar -e "DELETE FROM wp_options WHERE option_name='rewrite_rules'"
   ```

### Bug fix: `const NAMESPACE` (sp-infra commit)

- `class-rest-controller.php`: `const NAMESPACE` --> `const API_NAMESPACE`
- `namespace` PHP reserved keyword -- PHP 8-ban fatal error
- WP error handler elnyelte, plugin csendben nem regisztralodott
- Fix utan: route regisztralodott, endpoint mukodik

### Smoke test

```
http://benettcar.local/wp-json/spektra/v1/site --> 200 OK
Response: {"site":[],"navigation":[],"pages":[]}
```

### sp-infra commit

`49d5a96` -- `feat: link-plugin.ps1 real impl + fix NAMESPACE reserved keyword in rest-controller (P4.3)`

### Statusz

Kesz.

---

## #4 -- Client overlay linking (2026-04-05) -- P4.4

**Cel:** `spektra-api.php` fallback pathjan (`__DIR__ . '/../spektra-config/config.php'`) a kliens config elerheto legyen.

### Mi tortent

Minden parancs **PowerShell**-bol futott (VS Code terminal).

1. **link-overlay.ps1 frissitve** (sp-infra commit `1f9db68`):
   - Scaffold lecserelve valos implementaciora
   - `New-Item -ItemType Junction` (nem SymbolicLink)
   - Target validation: ha junction letezik, ellenorzi, hogy helyes source-ra mutat-e
   - Ha rossz target: error (nem csereli csendben)

2. **Junction letrehozva** (PowerShell):
   ```powershell
   cd D:\Projects\spektra\sp-infra
   & .\scripts\link-overlay.ps1 -Client benettcar
   ```
   Eredmeny:
   ```
   .local\wp-runtimes\benettcar\wp-content\plugins\spektra-config
     --> D:\Projects\spektra\sp-clients\sp-benettcar\infra
   ```

3. **Junction ellenorzes**:
   ```powershell
   Get-Item "D:\Projects\spektra\.local\wp-runtimes\benettcar\wp-content\plugins\spektra-config" |
     Select-Object Mode, Name, Target | Format-List
   ```
   Eredmeny: `Mode: d----l`, `Target: ...sp-benettcar\infra`

4. **config.php elerheto fallback pathon**:
   ```powershell
   & "D:\Local\wamp\bin\php\php8.4.15\php.exe" -r "var_dump(file_exists('D:/Projects/spektra/.local/wp-runtimes/benettcar/wp-content/plugins/spektra-config/config.php'));"
   ```
   Eredmeny: `bool(true)`

### Ketlepcsos smoke test

| Ellenorzes | Eredmeny |
|---|---|
| debug.log letezik elotte? | Nem (nincs korabbi error) |
| active_plugins DB | `a:1:{i:0;s:27:"spektra-api/spektra-api.php";}` -- plugin aktiv |
| GET /wp-json/spektra/v1/site | 200 OK, `{"site":[],"navigation":[],"pages":[]}` |
| debug.log letrehozva utana? | Nem (nincs uj error) |

### Idempotencia teszt

```powershell
& .\scripts\link-overlay.ps1 -Client benettcar
# --> "Overlay junction already exists and points to correct source: ..."
```

### Architektura megjegyzes

Ez a symlink fallback a **jelenlegi runtime strategia**. Kesobb a `bootstrap.ps1` ugyanezt
a logikat hivja scriptbol, a `wp-config.php.tpl` + ENV-driven path a vegleges modell.

### sp-infra commit

`1f9db68` -- `feat: link-overlay.ps1 real impl -- Junction + target validation (P4.4)`

### Statusz

Kesz.

---

## #8 -- Client field registration verification (2026-04-05) -- P6.2

**Cel:** Verifikacio, hogy a 10 bc-* field group megjelenik a Fooldal szerkeszton.

### Ellenorzott elemek

**1. Front page setup:**
- `show_on_front`: `page`
- `page_on_front`: ID 7 ("Home", status: publish, slug: home)

**2. Field group registration:**
- 10/10 group regisztralva
- 10/10 group illeszkedik a front page post ID-hez (location rule: `page_type == front_page`)

**3. Config sections vs ACF groups alignment:**
- Config sections (10): bc-hero, bc-brand, bc-gallery, bc-services, bc-service, bc-about, bc-team, bc-assistance, bc-contact, bc-map
- ACF group slugs (10): bc-hero, bc-brand, bc-gallery, bc-services, bc-service, bc-about, bc-team, bc-assistance, bc-contact, bc-map
- Missing: NONE
- Extra: NONE

**4. Field type inventory (51 field, 8 unique type):**

| Type | Count |
|---|---|
| text | 28 |
| textarea | 6 |
| image | 2 |
| repeater | 8 |
| true_false | 1 |
| group | 2 |
| select | 3 |
| number | 1 |

**5. Per-group detail:**

| Group | Fields | Types |
|---|---|---|
| BC Hero | 8 | text(7), image(1) |
| BC Brand | 3 | text(1), textarea(1), repeater(1) |
| BC Gallery | 4 | text(2), true_false(1), repeater(1) |
| BC Services | 3 | text(2), repeater(1) |
| BC Service | 6 | text(2), textarea(1), repeater(2), group(1) |
| BC About | 9 | text(3), repeater(2), image(1), select(2) |
| BC Team | 4 | text(2), textarea(1), repeater(1) |
| BC Assistance | 6 | text(5), textarea(1) |
| BC Contact | 5 | text(2), textarea(1), group(1), select(1) |
| BC Map | 3 | text(2), number(1) |

### Verdict

ALL CHECKS PASS — nincs kodvaltozas, verifikacios lepes.

### Statusz

✅ Kesz.

---

## #9 -- Field definition integrity audit (2026-04-05) -- P6.3

**Cel:** Minden mezo type-specifikus property-k futasideju ellenorzese.

### Audit script

WP runtime-ban futtatott PHP script: 10 group, 51 top-level field + osszes sub_field.

### Ellenorzott property-k

| Mező típus | Ellenőrzött property-k |
|---|---|
| Alapvető (mind) | key, name, type, label |
| image | return_format=array |
| repeater | sub_fields count > 0, sub key/name/type |
| group | sub_fields count > 0, sub key/name/type |
| select | choices count ≥ 2, default_value |
| number | default_value |
| true_false | default_value |
| textarea | rows > 0 |
| image sub_field | return_format=array (repeater/group belsejében) |

### Eredmény

- **329 check futtatva**
- **329 PASS, 0 FAIL**
- Minden image field: return_format=array ✅
- Minden repeater: van sub_fields (1-5 sub) ✅
- Minden group: van sub_fields (3-8 sub) ✅
- Minden select: 2 choice + default ✅
- number: default_value=400 ✅
- true_false: default_value=0 ✅
- textarea: rows 2-4 ✅

### Verdict

ALL CHECKS PASS (329/329) — nincs kódváltozás, verifikációs lépés.

### Statusz

✅ Kész.

---

## #7 -- Phase 5 full smoke test (2026-04-05) -- P5.5

**Cel:** Phase 5 zaro teszt -- REST endpoint, preview, validacio, CORS, preflight, debug.log.

### Eredmenyek (18/18 PASS)

| # | Teszt | Eredmeny |
|---|---|---|
| 1 | GET /site → 200 | PASS |
| 2 | Valid JSON | PASS |
| 3 | Has site/navigation/pages keys | PASS (3x) |
| 4 | X-Spektra-Version: 0.1.0 | PASS |
| 5 | preview=true → 200 | PASS |
| 6 | Cache-Control: no-cache (preview) | PASS |
| 7 | preview=yes → 400 | PASS |
| 8 | Error code rest_invalid_param | PASS |
| 9 | CORS Allow-Origin (localhost:5174) | PASS |
| 10 | Vary: Origin | PASS |
| 11 | CORS disallowed (evil.com) → no header | PASS |
| 12 | OPTIONS → 204 | PASS |
| 13 | Preflight Allow-Origin | PASS |
| 14 | Allow-Methods: GET, OPTIONS | PASS |
| 15 | Allow-Headers: Content-Type, Authorization | PASS |
| 16 | debug.log: 0 PHP errors | PASS |

### Phase 5 osszefoglalas

| Lepes | Commit | Lenyeg |
|---|---|---|
| P5.1 | `9b0321f` | spektra-api.php finalize (API_URL, clean comments) |
| P5.2 | `0f7a129` | rest-controller (schema, validate preview, version+cache headers) |
| P5.3 | `26a3f21` | CORS (origin whitelist, preflight 204, Vary, WP override) |
| P5.4 | — | Config loading verification (nincs kodvaltozas) |
| P5.5 | — | Zaro smoke test (18/18 PASS) |

### Statusz

Phase 5 KESZ.

---

## #6 -- Config loading verification (2026-04-05) -- P5.4

**Cel:** Explicit verifikacio, hogy a teljes plugin config lanca mukodik.

### Verifikalt elemek

| # | Ellenorzes | Eredmeny |
|---|---|---|
| 1 | `SPEKTRA_CLIENT_CONFIG` defined | YES |
| 2 | Tipusa tomb | YES |
| 3 | `client_slug` | `benettcar` |
| 3 | `client_name` | `Benett Car` |
| 3 | `allowed_origins` | `['http://localhost:5174']` |
| 3 | `site_defaults` | `lang=hu, title=Benett Car` |
| 3 | `sections` | 10 item (bc-hero ... bc-map) |
| 4 | ACF field groups | 10 group, 51 field |
| 5 | `SPEKTRA_API_VERSION` | `0.1.0` |
| 5 | `SPEKTRA_API_PATH` | `.../sp-infra/plugin/spektra-api/` |
| 5 | `SPEKTRA_API_URL` | `http://benettcar.local/wp-content/plugins/spektra-api/` |

### Megjegyzes

- Nincs kodvaltozas -- ez tisztan verifikacios lepes
- PHP CLI-bol futott (`wp-load.php` require)
- A `SPEKTRA_API_PATH` a Junction forras utvonalat mutatja (sp-infra), a `SPEKTRA_API_URL` a WP-s utvonalat -- ez vart viselkedes

### Statusz

Kesz.

---

## #5 -- ACF install + verify (2026-04-05) -- P4.5

**Cel:** ACF Free telepites, 10 kliens field group betoltodesenek ellenorzese.

### Mi tortent

Minden parancs **PowerShell**-bol futott (VS Code terminal).

1. **ACF Free letoltve es kicsomagolva:**
   ```powershell
   $acfZip = "$env:TEMP\advanced-custom-fields.zip"
   Invoke-WebRequest -Uri "https://downloads.wordpress.org/plugin/advanced-custom-fields.latest-stable.zip" -OutFile $acfZip -UseBasicParsing
   $pluginsDir = "D:\Projects\spektra\.local\wp-runtimes\benettcar\wp-content\plugins"
   Expand-Archive -Path $acfZip -DestinationPath $pluginsDir -Force
   Remove-Item $acfZip -Force
   ```

2. **ACF aktivalas WP PHP API-bol** (NEM serialized MySQL hack):
   ```powershell
   # Egyszer hasznalatos script (utana torolve):
   & "D:\Local\wamp\bin\php\php8.4.15\php.exe" activate-acf.php
   ```
   A script `activate_plugin('advanced-custom-fields/acf.php')` WP fuggvenyt hasznalja.
   Eredmeny: `ACF activated successfully`
   Active plugins: `advanced-custom-fields/acf.php, spektra-api/spektra-api.php`

### Bug fix: `__DIR__` -> `WP_PLUGIN_DIR` (sp-infra commit)

- **Problema:** `__DIR__` PHP-ban Junction-on at a valos forras utvonalra old fel
  (`D:\Projects\spektra\sp-infra\plugin\spektra-api\`), nem a junction utvonalra
  (`wp-content/plugins/spektra-api/`). Igy a `__DIR__ . '/../spektra-config/'`
  sotetbe mutatott -- se config, se ACF field groups nem toltodott be.
- **Tenet:** Ez P4.4-bol oroklott latens bug volt. A 200 OK elfeddte, mert a
  placeholder response nem fugg a configtol.
- **Fix:** `WP_PLUGIN_DIR . '/spektra-config/config.php'` -- ez a tenyleges
  plugins mappat adja, ahol a `spektra-config` Junction el.
- Ugyanitt: ACF field group loader hozzaadva (`require_once field-groups.php`)

### Ketlepcsos smoke test

| Ellenorzes | Eredmeny |
|---|---|
| debug.log baseline | Letezett, 3 sor (auto-update + korabbi diag) |
| Endpoint | 200 OK, `{"site":[],"navigation":[],"pages":[]}` |
| debug.log uj hiba? | Nincs |
| [diag] ACF loaded | `yes` |
| [diag] Local field groups | `10` |

### ACF field group reszletek

| Field Group | Key | Fields |
|---|---|---|
| BC Hero | group_bc_hero | 8 |
| BC Brand | group_bc_brand | 3 |
| BC Gallery | group_bc_gallery | 4 |
| BC Services | group_bc_services | 3 |
| BC Service | group_bc_service | 6 |
| BC About | group_bc_about | 9 |
| BC Team | group_bc_team | 4 |
| BC Assistance | group_bc_assistance | 6 |
| BC Contact | group_bc_contact | 5 |
| BC Map | group_bc_map | 3 |
| **Osszesen** | | **51** |

### sp-infra commit

`43c4456` -- `fix: config path __DIR__ -> WP_PLUGIN_DIR + add ACF field group loader (P4.5)`

### Statusz

Kesz.
