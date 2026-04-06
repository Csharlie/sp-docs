# P4.3 Post-Mortem — Plugin Symlink + Activate

**Dátum:** 2026-04-05
**Lépés:** Phase 4.3 — Plugin symlink + activate
**Repók:** sp-infra (`49d5a96`), sp-docs (runtime-log #3)
**Időigény:** Jelentős — 3 külön bug, mindegyik csendben bukott

---

## Összefoglaló

A P4.3 célja egyszerűnek tűnt: a `sp-infra/plugin/spektra-api/` könyvtárat Junction-nal
belinkelni a WP runtime plugins mappájába, aktiválni a plugint, és ellenőrizni,
hogy a `/wp-json/spektra/v1/site` endpoint 200 OK-val válaszol.

A valóságban **3 különálló, egymásra épülő bug** került elő, mindegyik csendben (silent failure)
bukott, ami jelentősen megnehezítette a hibakeresést.

---

## A terv

```
1. link-plugin.ps1 scaffold → valós implementáció
2. Script futtatás → Junction létrehozás
3. Plugin aktiválás → MySQL UPDATE (active_plugins)
4. Smoke test → GET /wp-json/spektra/v1/site → 200 OK
```

---

## Ami működött elsőre

### Junction létrehozás (link-plugin.ps1)

A script scaffold-ot lecseréltük valós implementációra:

```powershell
# Lényegi rész
$PluginSource = Join-Path $WorkspaceRoot "sp-infra\plugin\spektra-api"
$PluginTarget = Join-Path $WorkspaceRoot ".local\wp-runtimes\$Client\wp-content\plugins\spektra-api"

New-Item -ItemType Junction -Path $PluginTarget -Target $PluginSource | Out-Null
```

**Döntés:** Junction, nem SymbolicLink.
- Windowson a Junction NEM igényel admin jogot
- Apache és PHP simán követi (FollowSymLinks az Apache vhost-ban)
- Idempotens: `Test-Path` guard, ha már létezik, skip

**Futtatás:**
```powershell
cd D:\Projects\spektra\sp-infra
& .\scripts\link-plugin.ps1 -Client benettcar
```

**Eredmény:** ✅ Junction létrejött, source fájlok elérhetők a target path-on.

**Ellenőrzés:**
```powershell
Get-Item "D:\Projects\spektra\.local\wp-runtimes\benettcar\wp-content\plugins\spektra-api" |
  Select-Object Mode, Name, Target
```
→ `Mode: d----l`, `Target: D:\Projects\spektra\sp-infra\plugin\spektra-api`

Ez a rész probléma nélkül ment.

---

## Bug #1 — Serialized string length mismatch

### Mi történt

A plugin aktiválást MySQL-en keresztül végeztük (nem WP admin, mert "activate early, fail early"):

```powershell
# HIBÁS — ELSŐ PRÓBÁLKOZÁS
echo 'UPDATE wp_options SET option_value=''a:1:{i:0;s:33:"spektra-api/spektra-api.php";}'' WHERE option_name=''active_plugins'';' | & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar
```

Az `s:33` a PHP serialized format string hossz jelölése. A `spektra-api/spektra-api.php` string
valójában **27 karakter**, nem 33.

### Miért nem derült ki azonnal

- A MySQL UPDATE sikeresen lefutott (0 error)
- A WP admin felületen a plugin "active"-nek tűnt a listában
- **DE** a PHP `unserialize()` csendben `false`-t adott vissza a hibás hossz miatt
- WordPress ilyenkor üres tömböt kap → egyetlen plugin sem töltődik be
- Nincs error log, nincs warning, nincs semmi jelzés

### Hogyan derült ki

A smoke test (`/wp-json/spektra/v1/site`) 404-et adott. Ellenőriztük a REST API index-et:

```powershell
(Invoke-WebRequest http://benettcar.local/wp-json/ -UseBasicParsing).Content | ConvertFrom-Json | Select-Object -ExpandProperty namespaces
```

Eredmény: csak `wp/v2` és `oembed/1.0` — a `spektra/v1` namespace HIÁNYZOTT.

Ez azt jelentette, hogy a plugin `rest_api_init` hook-ja nem futott le → a plugin nem töltődött be.

### Megoldás

```powershell
# JAVÍTOTT — s:27
echo 'UPDATE wp_options SET option_value=''a:1:{i:0;s:27:"spektra-api/spektra-api.php";}'' WHERE option_name=''active_plugins'';' | & "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar
```

**Tanulság:** PHP serialized format-ban az `s:N` mező **bájt-pontos** hosszt vár.
Ha nem stimmel, az egész `unserialize()` csendben meghiúsul. Nincs partial parse, nincs warning.
Ellenőrzés: `"spektra-api/spektra-api.php".Length` → 27.

---

## Bug #2 — `const NAMESPACE` — PHP reserved keyword

### Mi történt

A serialized hossz javítása után újra teszteltünk — **továbbra is 404**.
A REST index-ben (`/wp-json/`) továbbra sem jelent meg a `spektra/v1` namespace.

Ellenőriztük, hogy a plugin fájl betöltődik-e:

```powershell
# Kézi PHP syntax check
& "D:\Local\wamp\bin\php\php8.4.15\php.exe" -l "D:\Projects\spektra\sp-infra\plugin\spektra-api\includes\class-rest-controller.php"
```

Eredmény: `No syntax errors detected` — a PHP linter NEM jelzett hibát!

### A probléma gyökere

A `class-rest-controller.php` fájlban ez állt:

```php
class Rest_Controller {
    const NAMESPACE = 'spektra/v1';   // ← HIBÁS
    const ROUTE     = '/site';
```

A `namespace` PHP-ban **reserved keyword** (PHP 5.3 óta). Class context-ben `const NAMESPACE`
illegális, de a hibajelzés trükkös:

1. **`php -l` (lint)** — NEM jelzi, mert a linter bizonyos kontextusokban elengedi
2. **PHP 8 runtime** — **fatal error**-t dob (`syntax error, unexpected token "namespace"`)
3. **WordPress error handler** — a fatal error-t elkapja,
   a plugin load csendben megszakad, de a WP tovább fut mintha mi sem történt volna
4. **A plugin "aktív" marad** az `active_plugins` DB-ben, de a kódja SOHA nem fut le

### Miért volt nehéz megtalálni

- `php -l` nem jelzett hibát
- WP admin-ban a plugin aktívnak tűnt
- Nem volt error output a HTTP response-ban
- `WP_DEBUG_LOG=true` volt, de a log fájlt nem vizsgáltuk azonnal
- A tünet (404 a route-on) ugyanaz volt, mint a serialization bugnál

### Hogyan derült ki

Kizárásos alapon: ha a plugin aktív, a fájlok ott vannak, a syntax check OK, de a route
mégsem regisztrálódik — akkor valami a FUTTATÁSKOR bukik csendben.

A `class-rest-controller.php` forrást tüzetesebben átnézve feltűnt, hogy a `NAMESPACE`
konstans neve egybeesik a PHP reserved keyword-del.

### Megoldás

```php
// ELŐTTE (hibás)
const NAMESPACE = 'spektra/v1';
// ...
register_rest_route( self::NAMESPACE, self::ROUTE, ... );

// UTÁNA (javított)
const API_NAMESPACE = 'spektra/v1';
// ...
register_rest_route( self::API_NAMESPACE, self::ROUTE, ... );
```

**Commit:** `49d5a96` (sp-infra) — `feat: link-plugin.ps1 real impl + fix NAMESPACE reserved keyword`

**Tanulság:**
- PHP reserved keyword-ök nem mindig buknak `php -l`-en
- WP error handler elnyeli a fatal error-t plugin betöltéskor → a plugin "aktív" de nem fut
- Elnevezés: soha ne használj PHP reserved keyword-öt konstans névnek (`NAMESPACE`, `CLASS`, `FUNCTION`, stb.)
- Ha egy WP plugin "aktív" de a hook-jai nem futnak, első tipp: **fatal error a plugin betöltésekor**

---

## Bug #3 — Permalink struktúra (Plain → Pretty)

### Mi történt

Az előző két fix után ismét tesztelünk. A plugin mostanra tényleg betöltődött,
a REST index-ben (`/wp-json/`) megjelent a `spektra/v1` namespace.

**De** a `http://benettcar.local/wp-json/spektra/v1/site` még mindig problémás volt bizonyos
kontextusban. A WordPress Plain permalink struktúra (`?p=123`) mellett a REST API ugyan elérhető
a `/wp-json/` alap-útvonalon, de a rewrite rules nélkül az Apache nem irányítja helyen a kéréseket.

### Megoldás

```powershell
# Permalink struktúra átállítása /%postname%/-ra
& "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar -e "UPDATE wp_options SET option_value='/%postname%/' WHERE option_name='permalink_structure'"

# Rewrite rules flush (WP újragenerálja a következő kérésnél)
& "D:\Local\wamp\bin\mysql\mysql8.4.7\bin\mysql.exe" -u root sp_benettcar -e "DELETE FROM wp_options WHERE option_name='rewrite_rules'"
```

**Tanulság:** Friss WP install-nál a permalink struktúra "Plain" — REST API route-ok
működhetnek a `?rest_route=/spektra/v1/site` formában, de a pretty URL
(`/wp-json/spektra/v1/site`) csak "Post name" vagy egyéb pretty permalink mellett működik
megbízhatóan Apache-on (kellő `.htaccess` / rewrite rules).

---

## Végleges smoke test

Minden fix után:

```powershell
$r = Invoke-WebRequest http://benettcar.local/wp-json/spektra/v1/site -UseBasicParsing
$r.StatusCode   # → 200
$r.Content       # → {"site":[],"navigation":[],"pages":[]}
```

✅ **200 OK** — a válasz megfelel a SiteData placeholder contract-nak.

---

## Időrendi összefoglaló

| # | Lépés | Eredmény | Probléma |
|---|-------|----------|----------|
| 1 | link-plugin.ps1 valós implementáció | ✅ Elsőre OK | — |
| 2 | Junction létrehozás (script futtatás) | ✅ Elsőre OK | — |
| 3 | Junction ellenőrzés (Get-Item) | ✅ Elsőre OK | — |
| 4 | Plugin aktiválás MySQL-ből (s:33) | ❌ Silent fail | Serialized string hossz hibás |
| 5 | Smoke test → 404 | ❌ | Namespace missing from REST index |
| 6 | Debug: namespace listázás | 🔍 spektra/v1 hiányzik | Plugin nem töltődött be |
| 7 | Fix: s:27 | ⚠️ Részleges fix | String hossz javítva, de… |
| 8 | Smoke test → 404 | ❌ | Namespace még mindig hiányzik! |
| 9 | Debug: php -l | 🟢 "No syntax errors" | Linter nem jelzi a reserved keyword-öt |
| 10 | Debug: forrás átnézés | 🔍 `const NAMESPACE` | PHP reserved keyword! |
| 11 | Fix: NAMESPACE → API_NAMESPACE | ✅ Namespace megjelent | Route regisztráció sikeres |
| 12 | Permalink: Plain → /%postname%/ | ✅ | Rewrite rules flush |
| 13 | Smoke test → 200 OK | ✅ | `{"site":[],"navigation":[],"pages":[]}` |

---

## Tanulságok összefoglalva

### 1. PHP serialized format kegyetlen

Az `unserialize()` nem "best effort" — ha egyetlen bájt eltérés van a string
hossz mezőben, az egész parse meghiúsul, **warning nélkül**. Mindig számold ki
pontosan: `strlen("spektra-api/spektra-api.php")` → 27, nem 33.

### 2. WordPress elnyeli a fatal error-t plugin betöltéskor

Ha egy plugin `aktív` a DB-ben de a kódja fatal error-ral elszáll, a WP error handler
elkapja, a plugin csendben nem töltődik be, de a WP tovább fut. A plugin az admin felületen
továbbra is "Active" státuszúnak tűnik. **Diagnosztika**: ha a plugin hook-jai nem futnak
(route missing, shortcode missing, stb.), nézd meg a `wp-content/debug.log`-ot.

### 3. `php -l` nem mindenható

A PHP linter (syntax check) bizonyos reserved keyword hibákat nem jelzi —
különösen class konstans kontextusban. Ne bízz kizárólag a linterben;
ha runtime anomáliát tapasztalsz, olvasd végig a forrást manuálisan.

### 4. PHP reserved keyword-ök listája (amit SOHA ne használj konstans névnek)

```
abstract, and, array, as, break, callable, case, catch, class, clone,
const, continue, declare, default, die, do, echo, else, elseif, empty,
enddeclare, endfor, endforeach, endif, endswitch, endwhile, eval, exit,
extends, final, finally, fn, for, foreach, function, global, goto, if,
implements, include, include_once, instanceof, insteadof, interface,
isset, list, match, NAMESPACE, new, or, print, private, protected,
public, readonly, require, require_once, return, static, switch,
throw, trait, try, unset, use, var, while, xor, yield
```

### 5. "Activate early, fail early" a helyes megközelítés

Pont azért aktiváltuk a plugint rögtön a Junction után (nem vártunk Phase 5-re),
hogy **most** derüljenek ki a problémák, ne 5 lépéssel később. Ez bevált:
a 3 bug így elkülönítve, lépésről lépésre volt debuggolható.

### 6. Junction Windows-on megbízható

A `New-Item -ItemType Junction` admin jog nélkül működik, Apache FollowSymLinks
követi, PHP `file_exists()` / `require` simán feloldja. Produkciós plugin
fejlesztéshez tökéletes.

---

## Érintett fájlok

| Fájl | Repo | Változás |
|------|------|----------|
| `scripts/link-plugin.ps1` | sp-infra | scaffold → valós implementáció |
| `plugin/spektra-api/includes/class-rest-controller.php` | sp-infra | `const NAMESPACE` → `const API_NAMESPACE` |
| `runtime-log.md` (#3) | sp-docs | P4.3 parancsok + fix dokumentáció |
| `docs/bootstrap-infra-log.md` (#12) | sp-infra | P4.3 log bejegyzés |

## Commitok

| Repo | Hash | Leírás |
|------|------|--------|
| sp-infra | `49d5a96` | feat: link-plugin.ps1 real impl + NAMESPACE fix (P4.3) |
| sp-infra | `38013dc` | docs: bootstrap-log #12 (P4.3) |
| sp-docs | `a93a2bc` | docs: runtime-log #3 (P4.3) |
