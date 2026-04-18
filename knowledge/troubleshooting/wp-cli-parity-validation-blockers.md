---
title: WP-CLI + Parity Validation — Környezeti Blockerek
status: draft
type: troubleshooting
phase_introduced: "12.4"
canonical: true
tags: [wp-cli, wamp, seed, parity, validation, environment]
last_updated: 2026-04-18
---

# WP-CLI + Parity Validation — Környezeti Blockerek

A P12.4 parity validation során felmerült akadályok és megoldásaik, hogy legközelebb ne kelljen újra felfedezni.

---

## B1 — Docker nem elérhető

**Tünet:** `docker` not recognized. A `sp-infra/docker/docker-compose.yml` létezik, de Docker Desktop nincs telepítve / nincs PATH-on.

**Megoldás:** A dev WP environment WAMP-on fut, nem Docker-ben.

**Tanulság:** A dev environment NEM feltétlenül a sp-infra docker-compose alapú. Először mindig kérdezd meg, hol fut a WP.

---

## B2 — PHP és WP-CLI nem PATH-on

**Tünet:** `where.exe php` és `where.exe wp` üres. Se `C:\wamp64`, se `C:\xampp` nem létezik.

**Megoldás:**
- WAMP: `D:\Local\wamp`
- PHP: `D:\Local\wamp\bin\php\php8.3.28\php.exe`
- WP-CLI: `D:\Local\wamp\wp-cli.phar`

**Hívás mindig így:**
```powershell
$php = "D:\Local\wamp\bin\php\php8.3.28\php.exe"
$wp  = "D:\Local\wamp\wp-cli.phar"
& $php $wp --path="..." <command>
```

**Tanulság:** WAMP custom elérési úton van. Ezeket fejben kell tartani — ne keresgélj, hanem használd a fenti path-eket.

---

## B3 — WordPress site nem a default WAMP www-ben

**Tünet:** `D:\Local\wamp\www` üres (nincsenek WP site-ok). A `wp-config.php` keresés más helyre vezet.

**Megoldás:** A WP runtime-ok itt vannak:
```
D:\Projects\spektra\.local\wp-runtimes\benettcar\
D:\Projects\spektra\.local\wp-runtimes\exotica\
```

**WP-CLI path argumentum:**
```powershell
$wpPath = "D:\Projects\spektra\.local\wp-runtimes\exotica"
& $php $wp --path=$wpPath eval "echo 'ok';"
```

**Site URL-ek:**
- Exotica: `http://exotica.local`
- BenettCar: (valószínűleg `http://benettcar.local`)

**Tanulság:** `--path` mindig a `.local/wp-runtimes/<client>` mappára kell mutasson.

---

## B4 — REST API fatal error WP-CLI-ből

**Tünet:** `rest_do_request()` hívás WP-CLI `eval`-ból fatal error-t dob.

**Ok:** A `rest_api_init` action NEM fut le automatikusan WP-CLI kontextusban. A Spektra REST route csak akkor regisztrálódik, ha ez az action lefut.

**Megoldás:** Mindig hívd meg manuálisan:
```php
do_action('rest_api_init');
$req = new WP_REST_Request('GET', '/spektra/v1/site');
$res = rest_do_request($req);
```

**Alternatíva:** Használd a `sp-infra/seed/verify-endpoint.php` scriptet, ami ezt már helyesen csinálja.

**Tanulság:** WP-CLI `eval` és `eval-file` NEM triggereli a `rest_api_init`-et. Mindig explicit `do_action('rest_api_init')` kell a REST hívás előtt.

---

## B5 — Seed import: összes update_field FAIL [RESOLVED]

**Tünet:** `wp eval-file import-seed.php seed.json` → 35 FAIL, 0 sikeres field. De manuális `update_field('eb_about_values_text', 'test', 4)` → `true`.

**Root cause:** `update_field()` delegates to WordPress core `update_metadata()`, which returns `false` when the new value is identical to the stored value ("no-op"). The import script treated every `false` return as an error — but on a second import (where seed data is already current) **all** fields return `false` because nothing changed.

**Fix (sp-infra commit):** When `update_field()` returns `false`, read the stored value back with `get_field($key, $post_id, false)` and compare via `field_values_match()`. If values match → "unchanged" (success). If mismatch → true failure.

New helper `field_values_match()` handles:
- Scalars: string-cast comparison (`(string) $a === (string) $b`)
- Images: stored as attachment ID (int > 0) vs seed `{url, alt}` object
- Arrays: JSON-encoded equality
- Null/empty edge cases

**Verbose flag:** `--verbose` must be passed as positional arg after `--` separator (not as WP-CLI flag):
```powershell
& $php $wp --path=$wpPath eval-file import-seed.php seed.json -- verbose
```

**Státusz:** RESOLVED — 37/37 field OK, 0 FAIL.

**Tanulság:** WordPress `update_metadata()` (and by extension ACF `update_field()`) returns `false` on no-change — this is by design, not an error. Any import/sync script must verify the stored value instead of relying on the return value.

---

## B6 — PowerShell UTF-8 encoding torzulás

**Tünet:** Magyar ékezetes karakterek a terminál output-ban olvashatatlanok (`FelelĹ's ĂˇllattartĂˇs`).

**Ok:** PowerShell alapértelmezett code page nem UTF-8.

**Megoldás:** Tartalmilag az adat helyes — a JSON-ben UTF-8. A terminál megjelenítés torzít, de az adat integritása nem sérül. Ha olvasható kimenet kell:
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

**Tanulság:** Ne a terminál kimeneten validálj UTF-8 szöveget. Használd a JSON-t fájlba irányítva, vagy a `--format=json` flag-et.

---

## Összefoglaló Cheat Sheet

```powershell
# ── Környezet ──
$php    = "D:\Local\wamp\bin\php\php8.3.28\php.exe"
$wp     = "D:\Local\wamp\wp-cli.phar"
$wpEx   = "D:\Projects\spektra\.local\wp-runtimes\exotica"
$wpBc   = "D:\Projects\spektra\.local\wp-runtimes\benettcar"

# ── Alap WP-CLI ──
& $php $wp --path=$wpEx eval "echo 'ok';"
& $php $wp --path=$wpEx plugin list --fields=name,status --format=table

# ── REST endpoint lekérés ──
& $php $wp --path=$wpEx eval "do_action('rest_api_init'); `$req = new WP_REST_Request('GET', '/spektra/v1/site'); `$res = rest_do_request(`$req); echo json_encode(`$res->get_data(), JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);"

# ── Seed export (sp-exotica) ──
cd D:\Projects\spektra\sp-clients\sp-exotica
pnpm seed:export

# ── Egyedi field írás ──
& $php $wp --path=$wpEx eval "update_field('eb_about_values_text', 'V1' . chr(10) . 'V2', 4);"

# ── Field olvasás ──
& $php $wp --path=$wpEx eval "var_dump(get_field('eb_about_values_text', 4));"

# ── ACF field groups ellenőrzés ──
& $php $wp --path=$wpEx eval "`$groups = acf_get_field_groups(); foreach (`$groups as `$g) { echo `$g['key'] . chr(10); }"
```
