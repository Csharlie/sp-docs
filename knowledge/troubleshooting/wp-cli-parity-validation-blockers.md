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

## B5 — Seed import: összes update_field FAIL

**Tünet:** `wp eval-file import-seed.php seed.json` → 35 FAIL, 0 sikeres field. De manuális `update_field('eb_about_values_text', 'test', 4)` → `true`.

**Ok:** Nem teljesen diagnosztizált. Valószínű okok:
1. Az `import-seed.php` a `$result === false` check-et használja, ami ACF 6.x-ben nem megbízható (ha az érték nem változik, ACF `false`-t adhat)
2. A `--verbose` flag az `import-seed.php`-ban nincs support-olva (error-t dob) — `--verbose` nem WP-CLI natív flag, hanem az import script saját paramétere, de az `eval-file` nem adja át
3. Lehetséges, hogy az ACF field registry és a field key mismatch okozza a FAIL-t batch import-nál

**Workaround:** Közvetlen `wp eval` + `update_field()` hívás működik:
```powershell
& $php $wp --path=$wpPath eval "update_field('eb_about_values_text', 'V1\nV2\nV3', 4);"
```

**Státusz:** NEM diagnosztizált — külön investigation szükséges. Nem P12.4-specifikus, minden field-re FAIL.

**Tanulság:** Az `import-seed.php` batch import nem megbízható. Egyedi `update_field` hívások működnek. A seed import script-et felül kell vizsgálni.

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
