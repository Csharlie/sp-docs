# P4.5 Post-Mortem — ACF Install + WP_PLUGIN_DIR Bug

**Dátum:** 2026-04-05
**Lépés:** Phase 4.5 — ACF install + verify
**Repók:** sp-infra (`43c4456`), sp-docs (runtime-log #5)
**Időigény:** Jelentős — 1 latens bug (P4.4-ből örökölt), 1 aktiválási kerülőút

---

## Összefoglaló

A P4.5 célja: ACF Free telepítése, aktiválás, és a 10 kliens field group (`bc-*`)
betöltődésének igazolása. A telepítés és kicsomagolás elsőre ment, de az aktiválás
és a field group verifikáció közben két különböző probléma jött elő:

1. **WP admin login nem működött PowerShell-ből** — cookie/session kezelési probléma
2. **`__DIR__` Junction path feloldás** — a config.php és az ACF loader sem talált rá az overlay-re

A második bug különösen alattomos volt, mert **P4.4 óta ott volt csendben**,
de a placeholder response elfeddte.

---

## A terv

```
1. ACF Free letöltés wordpress.org-ról
2. Kicsomagolás wp-content/plugins-be
3. Aktiválás (NEM serialized hack — P4.3 tanulság)
4. Field group-ok betöltődésének ellenőrzése
5. Kétlépcsős smoke test
```

---

## Ami működött elsőre

### ACF letöltés + kicsomagolás

```powershell
$acfZip = "$env:TEMP\advanced-custom-fields.zip"
Invoke-WebRequest -Uri "https://downloads.wordpress.org/plugin/advanced-custom-fields.latest-stable.zip" `
  -OutFile $acfZip -UseBasicParsing
$pluginsDir = "D:\Projects\spektra\.local\wp-runtimes\benettcar\wp-content\plugins"
Expand-Archive -Path $acfZip -DestinationPath $pluginsDir -Force
Remove-Item $acfZip -Force
```

Eredmény: `advanced-custom-fields/acf.php` a helyén. ✅

---

## Probléma #1 — WP admin login PowerShell-ből

### Mi történt

A P4.3 tanulság alapján a döntés: **ne MySQL serialized hack**-kel aktiváljunk,
hanem a WP saját mechanizmusán keresztül. Első próbálkozás: HTTP POST a login formra,
majd a plugins.php activate linkre.

```powershell
# 1. próba: SessionVariable
$loginBody = @{
    log = 'admin'; pwd = 'admin'; 'wp-submit' = 'Log In'
    redirect_to = 'http://benettcar.local/wp-admin/'; testcookie = '1'
}
Invoke-WebRequest -Uri "http://benettcar.local/wp-login.php" `
  -Method POST -Body $loginBody -SessionVariable session -UseBasicParsing
```

**Eredmény:** `Session cookies: 0` — a session nem tartott meg semmit.

```powershell
# 2. próba: explicit CookieContainer + kétlépcsős login
$cc = New-Object System.Net.CookieContainer
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
$session.Cookies = $cc

# Login page GET (cookie-k begyűjtése)
Invoke-WebRequest -Uri "http://benettcar.local/wp-login.php" -WebSession $session -UseBasicParsing
# Login POST
Invoke-WebRequest -Uri "http://benettcar.local/wp-login.php" -Method POST `
  -Body $loginBody -WebSession $session -UseBasicParsing
```

**Eredmény:** A POST visszairányított a login page-re `?reauth=1` paraméterrel.
A WP nem fogadta el a session-t.

### Miért nem működött

PowerShell 5.1 `Invoke-WebRequest` + `.local` domain + WP cookie kezelés = problémás kombináció:
- A WP `testcookie` mechanizmusa elvárja, hogy a böngésző a GET request cookie-ját küldje vissza
- PowerShell 5.1 `WebRequestSession` cookie kezelése a `.local` TLD-vel nem megbízható
- A `redirect_to` chain közben a cookie-k elveszhetnek

### Megoldás: PHP CLI + WP API

Ahelyett, hogy tovább kísérletezgettünk volna a PowerShell HTTP session-nel,
egyszer használatos PHP script-et írtunk a WP runtime-ba:

```php
<?php
define( 'WP_USE_THEMES', false );
require_once __DIR__ . '/wp-load.php';
require_once ABSPATH . 'wp-admin/includes/plugin.php';

$result = activate_plugin( 'advanced-custom-fields/acf.php' );

if ( is_wp_error( $result ) ) {
    echo 'ERROR: ' . $result->get_error_message() . PHP_EOL;
    exit( 1 );
} else {
    echo 'ACF activated successfully' . PHP_EOL;
    $active = get_option( 'active_plugins' );
    echo 'Active plugins: ' . implode( ', ', $active ) . PHP_EOL;
}
```

```powershell
& "D:\Local\wamp\bin\php\php8.4.15\php.exe" activate-acf.php
# → ACF activated successfully
# → Active plugins: advanced-custom-fields/acf.php, spektra-api/spektra-api.php
```

Script ezután törölve.

**Tanulság:**
- PHP CLI + `wp-load.php` + `activate_plugin()` megbízhatóbb, mint HTTP session hack
- Pontosan ugyanezt csinálja a WP-CLI is (`wp plugin activate`)
- Egyszer használatos script pattern: létrehoz → futtat → töröl — tiszta, nincs maradék

---

## Probléma #2 — `__DIR__` Junction path feloldása (latens P4.4 bug)

### Ez volt a nagy.

Az ACF aktiválás után a diag log ezt mutatta:

```
[spektra-diag] ACF loaded: yes
```

De a field group ellenőrzés:

```
Field groups registered: 0
```

### Első nyomozás: hook timing

Feltételezés: talán az `acf/init` hook nem fut le CLI kontextusban.

```php
// check-acf.php
if ( ! did_action( 'acf/init' ) ) {
    echo 'acf/init has NOT fired';
} else {
    echo 'acf/init fired, count: ' . did_action( 'acf/init' );
}
```

Eredmény:
```
init already fired, count: 1
acf/init fired, count: 1
Local field groups: 0
```

**Kizárva:** a hook lefutott, tehát nem timing probléma.

### Második nyomozás: file path feloldás

Diagnosztika hozzáadva a `spektra-api.php`-ba:

```php
if ( file_exists( $spektra_acf_path ) ) {
    require_once $spektra_acf_path;
    error_log( '[spektra-diag] field-groups.php loaded from: ' . $spektra_acf_path );
} else {
    error_log( '[spektra-diag] field-groups.php NOT FOUND at: ' . $spektra_acf_path );
}
```

HTTP request után a debug.log:

```
[spektra-diag] field-groups.php NOT FOUND at:
  D:\Projects\spektra\sp-infra\plugin\spektra-api/../spektra-config/acf/field-groups.php
```

### A probléma gyökere

A `spektra-api.php` 31. sora (P3.1 óta):

```php
$spektra_config_path = getenv('SPEKTRA_CLIENT_CONFIG')
    ?: __DIR__ . '/../spektra-config/config.php';
```

A `__DIR__` PHP-ban **a fájl valós, fizikai helyét** adja vissza,
nem a junction/symlink útvonalat. Windowson a Junction egy NTFS reparse point,
de a PHP `__DIR__` ezen **átlát**:

```
__DIR__ → D:\Projects\spektra\sp-infra\plugin\spektra-api\
                                        ↑ valós forrás hely

nem → D:\Projects\spektra\.local\wp-runtimes\benettcar\wp-content\plugins\spektra-api\
                                                                   ↑ junction path
```

Ezért:
```
__DIR__ . '/../spektra-config/'
→ D:\Projects\spektra\sp-infra\plugin\spektra-config\   ← NEM LÉTEZIK!

Kellene:
→ D:\...\wp-content\plugins\spektra-config\               ← itt van a junction
```

### Miért nem derült ki P4.4-ben

P4.4-ben a Junction ellenőrzés ezt mutatta:
- `file_exists` PHP CLI-ből → `bool(true)` (a CLI a helyes path-ot adta meg kézzel)
- Endpoint → 200 OK

**DE** a 200 OK megtévesztő volt! A `spektra-api.php`-ban a config loading guard:

```php
if ( file_exists( $spektra_config_path ) ) {
    $spektra_config = require $spektra_config_path;
    // ...
}
// Ha nem létezik, $spektra_config = [] marad
define( 'SPEKTRA_CLIENT_CONFIG', $spektra_config );
```

A `file_exists` csendben `false`-t adott → `$spektra_config = []` → a plugin
továbbra is működött, mert a placeholder response nem függött a config tartalmától.

**Tehát a config.php P4.4 óta SOHA nem töltődött be.** A P4.4 smoke test
kizárólag azt mutatta, hogy a plugin nem crashelt — nem azt, hogy a config
tényleg betöltődött.

### Megoldás

```php
// ELŐTTE (hibás — P3.1 óta)
$spektra_config_path = getenv('SPEKTRA_CLIENT_CONFIG')
    ?: __DIR__ . '/../spektra-config/config.php';

// UTÁNA (javított — P4.5)
$spektra_config_path = getenv('SPEKTRA_CLIENT_CONFIG')
    ?: WP_PLUGIN_DIR . '/spektra-config/config.php';
```

A `WP_PLUGIN_DIR` a WordPress által definiált konstans:
`wp-content/plugins/` abszolút path. Ez mindig a tényleges plugins mappát adja,
függetlenül attól, hogy a hívó fájl honnan töltődött be.

**Commit:** `43c4456` (sp-infra)

### Javítás utáni debug.log

```
[spektra-diag] field-groups.php loaded from:
  D:\Projects\spektra\.local\wp-runtimes\benettcar/wp-content/plugins/spektra-config/acf/field-groups.php
[spektra-diag] ACF loaded: yes
[spektra-diag] Local field groups: 10
```

---

## Végleges smoke test

Diagnosztika eltávolítása után, tiszta kóddal:

```powershell
$r = Invoke-WebRequest http://benettcar.local/wp-json/spektra/v1/site -UseBasicParsing
$r.StatusCode   # → 200
$r.Content       # → {"site":[],"navigation":[],"pages":[]}
```

debug.log: **üres** — nincs fatal, nincs warning.

Field group részletes lista (PHP CLI-ből):

```
Field groups: 10
  BC Hero (group_bc_hero) - 8 fields
  BC Brand (group_bc_brand) - 3 fields
  BC Gallery (group_bc_gallery) - 4 fields
  BC Services (group_bc_services) - 3 fields
  BC Service (group_bc_service) - 6 fields
  BC About (group_bc_about) - 9 fields
  BC Team (group_bc_team) - 4 fields
  BC Assistance (group_bc_assistance) - 6 fields
  BC Contact (group_bc_contact) - 5 fields
  BC Map (group_bc_map) - 3 fields
```

10 field group, **51 mező összesen.** ✅

---

## Időrendi összefoglaló

| # | Lépés | Eredmény | Probléma |
|---|-------|----------|----------|
| 1 | ACF zip letöltés | ✅ Elsőre OK | — |
| 2 | Kicsomagolás plugins-be | ✅ Elsőre OK | — |
| 3 | WP admin login (PowerShell HTTP, 1. próba) | ❌ | Session cookies: 0 |
| 4 | WP admin login (CookieContainer, 2. próba) | ❌ | Redirect → ?reauth=1 |
| 5 | Aktiválás: PHP CLI + activate_plugin() | ✅ | Kerülőút működött |
| 6 | Diag log: ACF loaded: yes | ✅ | ACF betöltődött |
| 7 | Field group count | ❌ 0 | Valami nem stimmel |
| 8 | Debug: acf/init hook lefutott? | ✅ Igen | Nem hook timing |
| 9 | Debug: file path ellenőrzés | 🔍 | config.php és acf path létezik a fájlrendszerben |
| 10 | Debug: error_log hozzáadva loader-hez | 🔍 | `NOT FOUND` a pathon |
| 11 | Felismerés: __DIR__ a valós source-ra old | 💡 | Junction-ok nem transzparensek __DIR__-nek |
| 12 | Felismerés: config.php sem töltődött be P4.4 óta | 💡 | Latens bug! |
| 13 | Fix: __DIR__ → WP_PLUGIN_DIR | ✅ | Path most helyes |
| 14 | Field groups: 10 | ✅ | 51 mező összesen |
| 15 | Diag eltávolítás, végső smoke test | ✅ | 200 OK, debug.log üres |

---

## Tanulságok összefoglalva

### 1. `__DIR__` nem Junction-safe

PHP-ben a `__DIR__` mindig a **fizikai fájl helyét** adja vissza, nem a symlink/junction
path-ot. Ha a plugin Junction-nal van linkelve, a `__DIR__` a source repo-ba mutat
(pl. `sp-infra/plugin/spektra-api/`), nem a `wp-content/plugins/spektra-api/`-ba.

**Relatív hivatkozások (`../`) ezért nem működnek cross-junction.**

Megoldás: `WP_PLUGIN_DIR` konstans használata, ami a WordPress által definiált
abszolút path a plugins mappához.

### 2. Placeholder response elfedi a tényleges hibákat

A `{"site":[],"navigation":[],"pages":[]}` 200 OK válasz ugyanúgy nézett ki:
- config betöltődött, de nincs tartalom
- config NEM töltődött be, guard `[]`-t adott

**Tanulság:** Placeholder fázisban a smoke test nem elégséges.
A config betöltődését explicit kell ellenőrizni (diag log, vagy a config tartalmának
megjelenítése a válaszban).

### 3. "Kétlépcsős smoke test" duplán kifizetődött

A P4.4-hez képest a P4.5-ben megkövetelt kétlépcsős teszt (debug.log + diag log)
pont azt a bugot hozta felszínre, amit a sima endpoint teszt elfedett volna.

### 4. WP admin HTTP session PowerShell 5.1-ből nem megbízható

A `.local` domain + WP cookie mechanizmus + PowerShell 5.1 `Invoke-WebRequest`
kombináció nem tartja meg a session cookie-kat megbízhatóan.

**Kerülőút:** PHP CLI + `wp-load.php` + WP API függvények. Ez azonos azzal,
amit a WP-CLI csinál — megbízható, scriptelhető, nincs cookie kezelés.

### 5. Latens bugok a legveszélyesebbek

A `__DIR__` bug P3.1 óta ott volt a kódban (a fallback path definíciójában).
P4.4-ben a Junction létrehozásakor életbe lépett, de csendben bukott.
Csak P4.5-ben, az ACF field group verifikációnál derült ki, mert ott
a 0 count egyértelmű jel volt.

**Tanulság:** Ha egy lépés sikerül (200 OK), de az adott lépés
lényegi hatása (config tartalom, field group regisztráció) nem volt
explicit tesztelve, a "sikerült" hamis volt.

### 6. Junction és `__DIR__` referencia más nyelveknél

| Nyelv/Eszköz | `__DIR__` / equivalent | Junction átlátás |
|---|---|---|
| PHP | `__DIR__` | **Igen** — valós path-ot ad |
| Node.js | `__dirname` | **Igen** — valós path-ot ad |
| Python | `__file__` | **Igen** — valós path-ot ad |
| Apache | FollowSymLinks | **Nem** — a junction path-ot követi |
| PowerShell | `$PSScriptRoot` | **Igen** — valós path-ot ad |

Vagyis: a legtöbb runtime "átlát" a junction-ön, az Apache nem.
Relatív path hivatkozásoknál ez mindig potenciális buktató.

---

## Érintett fájlok

| Fájl | Repo | Változás |
|------|------|----------|
| `plugin/spektra-api/spektra-api.php` | sp-infra | `__DIR__` → `WP_PLUGIN_DIR` + ACF loader |
| `runtime-log.md` (#5) | sp-docs | P4.5 parancsok + bug leírás |
| `docs/bootstrap-infra-log.md` (#14) | sp-infra | P4.5 log bejegyzés |

## Commitok

| Repo | Hash | Leírás |
|------|------|--------|
| sp-infra | `43c4456` | fix: config path __DIR__ -> WP_PLUGIN_DIR + ACF loader (P4.5) |
| sp-infra | `0621c7a` | docs: bootstrap-log #14 (P4.5) |
| sp-docs | `b858a4e` | docs: runtime-log #5 (P4.5) |
