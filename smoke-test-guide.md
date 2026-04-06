# Smoke Test — Mi ez, hogyan működik, miért használjuk

> Célközönség: junior fejlesztők, senior fejlesztők, külső kontribútorok.
> Utolsó frissítés: 2026-04-05

---

## Tartalomjegyzék

1. [Mi az a smoke test?](#mi-az-a-smoke-test)
2. [Honnan jön a név?](#honnan-jön-a-név)
3. [Hogyan működik?](#hogyan-működik)
4. [Mire jó és mire nem?](#mire-jó-és-mire-nem)
5. [Hol szokták alkalmazni?](#hol-szokták-alkalmazni)
6. [Miért jó a Spektrára?](#miért-jó-a-spektrára)
7. [Konkrét Spektra példák](#konkrét-spektra-példák)
8. [Kétlépcsős smoke test](#kétlépcsős-smoke-test)
9. [Hogyan fut nálunk?](#hogyan-fut-nálunk)
10. [Összefoglaló](#összefoglaló)

---

## Mi az a smoke test?

A smoke test a legegyszerűbb kérdést teszi fel:

> **„Működik egyáltalán?"**

Nem azt vizsgálja, hogy minden funkció tökéletesen viselkedik-e. Nem ellenőrzi a szélső
eseteket, a hibakezelést, a teljesítményt. Egyetlen célja van: meggyőződni arról, hogy
a rendszer (vagy egy komponens) **elindul, válaszol és nem dől össze**.

Analógia: amikor megnyomod a villanykapcsolót, nem a fényforrás színhőmérsékletét
méred — csak azt nézed, hogy **felkapcsol-e a lámpa**.

---

## Honnan jön a név?

Az elektronikából ered. Amikor egy új áramköri lapot először csatlakoztattak az áramra,
a mérnökök arra figyeltek: **száll-e füst belőle** (smoke). Ha nem füstölt, az azt
jelentette, hogy az alap bekötés rendben van, és érdemes folytatni a részletes tesztelést.
Ha füstölt — azonnal ki a konnektorból, hibajavítás.

A szoftverfejlesztésben ugyanez a logika:

| Elektronika | Szoftver |
|---|---|
| Dugd be, nézd, füstöl-e | Indítsd el, nézd, crashel-e |
| Nincs füst → folytasd a tesztelést | 200 OK → folytasd a fejlesztést |
| Füstöl → ne is mérd tovább | 500/Fatal → ne is nyúlj hozzá, javítsd |

---

## Hogyan működik?

Egy smoke test tipikusan **3 lépésből** áll:

### 1. Kérés küldése

Meghívsz egy endpointot, betöltesz egy oldalt, futtatod a programot — valami olyat
csinálsz, amit az éles rendszer is csinálna.

```
GET http://benettcar.local/wp-json/spektra/v1/site
```

### 2. Válasz ellenőrzése

Megnézed, hogy a válasz **létezik és értelmes**:

- HTTP státuszkód: `200 OK` (nem `500`, nem `404`)
- A válasz formátuma helyes (JSON, nem HTML hibaoldal)
- Az alapstruktúra megvan (van `site`, `navigation`, `pages` kulcs)

### 3. Mellékhatások ellenőrzése

Megnézed, hogy a kérés **nem okozott hibát**:

- Nincs új bejegyzés a `debug.log`-ban
- Nincs PHP fatal error, warning, notice
- A rendszer továbbra is válaszol

Ha mind a három pont zöld → **a smoke test sikeres**.

---

## Mire jó és mire nem?

### ✅ Mire jó

| Cél | Példa |
|---|---|
| **Gyors sanity check** | Épp deploy-oltam, működik-e az oldal? |
| **Regresszió-szűrő** | Módosítottam a configot, nem törtem el az API-t? |
| **Build verification** | Új verzió után az alkalmazás egyáltalán elindul? |
| **Integráció ellenőrzés** | Két komponens össze van kötve, kommunikálnak? |

### ❌ Mire NEM jó

| Nem cél | Miért nem |
|---|---|
| **Funkcionális tesztelés** | Nem vizsgálja, hogy a válasz tartalma helyese |
| **Edge case-ek** | Nem teszteli a 0 elem, null input, timeout eseteket |
| **Teljesítmény** | Nem méri a válaszidőt, memóriát, terhelhetőséget |
| **Biztonsági audit** | Nem vizsgálja az XSS-t, injection-t, auth bypass-t |

A smoke test nem helyettesíti a többi tesztelési szintet. Azt mondja meg: **érdemes-e
egyáltalán elkezdeni a részletes tesztelést**, vagy előbb van egy alapvető probléma.

---

## Hol szokták alkalmazni?

A smoke test univerzális eszköz. Bárhol megjelenik, ahol van **build → deploy → futtatás**
ciklus:

### Backend / API fejlesztés

```bash
# REST API smoke test
curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health
# Elvárás: 200
```

Használják: Node.js (Express, Fastify), PHP (Laravel, WordPress), Python (Django, Flask),
Go, Rust — nyelv- és framework-független.

### Frontend / SPA

```bash
# Vite dev szerver elindul-e?
curl -s -o /dev/null -w "%{http_code}" http://localhost:5174
# Elvárás: 200
```

React, Vue, Svelte, Angular — bármilyen frontend buildnél.

### CI/CD pipeline-ok

A legtöbb CI rendszerben (GitHub Actions, GitLab CI, Jenkins) a smoke test a pipeline
**első tesztlépése**. Ha elbukik, a pipeline azonnal leáll — nem futtatja a lassabb
unit/integration testeket.

```yaml
# GitHub Actions példa
- name: Smoke test
  run: |
    curl --fail http://localhost:8080/health
```

### WordPress plugin fejlesztés

WordPress-ben nincs beépített tesztrendszer. A smoke test különösen fontos, mert:

- A plugin aktiválás csendben bukhat (fatal error → auto-deactivate)
- A REST API regisztráció sok ponton eltörhet
- A `debug.log` az egyetlen megbízható hibajelző

A Spektra pontosan ebben a környezetben használja.

---

## Miért jó a Spektrára?

A Spektra architektúra három sajátosság miatt **különösen rá van szorulva** a smoke
testre:

### 1. Junction-alapú fájlrendszer

A Spektra plugin (`spektra-api`) és a kliens config (`spektra-config`) **Junction
linkeken** keresztül kerül a WordPress `wp-content/plugins/` mappájába. A fájlok fizikai
helye (`sp-infra/`, `sp-benettcar/`) és a WordPress által látott helye (`plugins/`)
különbözik.

Ez azt jelenti, hogy **a fájlok megléte nem garancia arra, hogy a WordPress is látja
őket**. A smoke test az egyetlen módja annak, hogy ellenőrizzük: a teljes lánc
(Junction → PHP autoload → WP hook → REST válasz) működik.

### 2. Placeholder válaszok elfedik a hibákat

A fejlesztés korai fázisaiban az endpointok placeholder (üres) válaszokat adnak:

```json
{"site": [], "navigation": [], "pages": []}
```

Ez a válasz **ugyanúgy néz ki**, ha:
- A config betöltődött, de nincs még tartalom
- A config **NEM** töltődött be, és a fallback `[]`-t adott

Egy sima `200 OK` ellenőrzés nem elég — a P4.5-ben pontosan ez a bug rejtőzött
hetekig. Ezért vezettük be a [kétlépcsős smoke testet](#kétlépcsős-smoke-test).

### 3. Nincs beépített teszt-framework

A Spektra jelenleg nem használ PHPUnit-ot, WP_Mock-ot, vagy bármilyen automatizált
tesztkeretrendszert a PHP oldalon. A smoke test az **egyetlen szisztematikus
ellenőrzés**, ami minden lépés után lefut.

---

## Konkrét Spektra példák

### Példa 1: Alapvető endpoint ellenőrzés (P4.3)

**Kontextus:** Épp most aktiváltuk a `spektra-api` plugint. Működik-e?

```powershell
$r = Invoke-WebRequest -Uri "http://benettcar.local/wp-json/spektra/v1/site" -UseBasicParsing
Write-Host "Status: $($r.StatusCode) Body: $($r.Content)"
```

**Elvárt kimenet:**
```
Status: 200 Body: {"site":[],"navigation":[],"pages":[]}
```

**Mit bizonyít:**
- WordPress fut ✅
- Plugin aktív ✅
- REST route regisztrálva ✅
- PHP autoload működik ✅
- Nincs fatal error ✅

**Mit NEM bizonyít:**
- Hogy a config tényleg betöltődik (ez a P4.5 lecke)
- Hogy az ACF field-ek regisztrálva vannak
- Hogy a válasz tartalma helyes

---

### Példa 2: debug.log ellenőrzés (P4.4)

**Kontextus:** Létrehoztuk a `spektra-config` Junction-t. Nincs-e mellékhatás?

```powershell
# 1. Endpoint
$r = Invoke-WebRequest -Uri "http://benettcar.local/wp-json/spektra/v1/site" -UseBasicParsing
Write-Host "Status: $($r.StatusCode)"

# 2. debug.log
$logPath = "D:\Projects\spektra\.local\wp-runtimes\benettcar\wp-content\debug.log"
if (Test-Path $logPath) {
    Write-Host "debug.log: $(Get-Content $logPath -Tail 5)"
} else {
    Write-Host "debug.log: clean"
}
```

**Elvárt kimenet:**
```
Status: 200
debug.log: clean
```

**Miért kell a debug.log is:**
Mert a PHP `require_once` csendben bukhat — nem ad 500-at, csak ír a logba.
A `200 OK` önmagában nem jelenti, hogy minden rendben van.

---

### Példa 3: ACF field group verifikáció (P4.5)

**Kontextus:** ACF-et telepítettük, 10 field group-ot regisztráltunk. Betöltődtek?

```powershell
& "D:\Local\wamp\bin\php\php8.4.15\php.exe" -r "
  require 'D:/Projects/spektra/.local/wp-runtimes/benettcar/wp-load.php';
  $groups = acf_get_local_field_groups();
  $fields = 0;
  foreach ($groups as \$g) {
      \$fields += count(acf_get_local_fields(\$g['key']));
  }
  echo count(\$groups) . ' groups, ' . \$fields . ' fields';
"
```

**Elvárt kimenet:**
```
10 groups, 51 fields
```

**Mit bizonyít:**
- ACF plugin aktív ✅
- `field-groups.php` betöltődik ✅
- Mind a 10 regisztráció lefut ✅
- A mezőszám pontos ✅

---

## Kétlépcsős smoke test

A P4.5 postmortem után bevezettük a **kétlépcsős** (two-step) smoke testet. Ez a sima
endpoint-ellenőrzésnél egy szinttel mélyebb:

### 1. lépcső: Endpoint + HTTP státusz

```
GET /wp-json/spektra/v1/site → 200 OK
```

Ez csak a „nem crashel" kérdésre válaszol.

### 2. lépcső: debug.log + diagnosztika

```
debug.log → nincs új error
[opcionális] diag output → config loaded, field groups = 10, stb.
```

Ez a „tényleg működik, amit kellene" kérdésre válaszol.

### Miért vezettük be?

| Fázis | Teszt típus | Eredmény | Valóság |
|---|---|---|---|
| P4.4 | Csak 1. lépcső | ✅ 200 OK | ❌ Config NEM töltődött be — bug rejtve maradt |
| P4.5 | Kétlépcsős | ❌ Field groups = 0 | ✅ Bug azonnal kiderült |

A kétlépcsős teszt a P4.5-ben pont azt a latens hibát hozta felszínre (`__DIR__`
Junction bug), amit a sima endpoint-teszt hónapokig elfedett volna.

**Szabály:** Minden lépés után kétlépcsős smoke testet futtatunk.

---

## Hogyan fut nálunk?

### Nem automatizált — manuálisan, az agent kezeli

A Spektra smoke test jelenleg **NEM** része egy CI/CD pipeline-nak, és **NEM** fut
automatikusan build-kor, commit-kor, vagy push-kor. Nincs hozzá regisztrált GitHub
Action, script parancs, vagy pre-commit hook.

A smoke testet **az AI agent (Copilot) futtatja manuálisan**, minden egyes v4 roadmap
lépés végén, a VS Code terminálban.

### Munkafolyamat

```
1. Kódmódosítás (pl. új konstans hozzáadása)
           │
           ▼
2. Smoke test – kétlépcsős
   ├── Endpoint: 200 OK?
   └── debug.log: üres?
           │
      ┌────┴────┐
      │OK       │FAIL
      ▼         ▼
3a. Commit   3b. Hibajavítás → vissza 2-re
      │
      ▼
4. Push + log entry
```

### Hol van dokumentálva?

A smoke test eredményei **két helyen** jelennek meg:

| Dokumentum | Mit tartalmaz | Példa |
|---|---|---|
| `sp-infra/docs/bootstrap-infra-log.md` | Infrastruktúra lépések smoke test eredményei | #14: "200 OK, debug.log üres" |
| `sp-docs/runtime-log.md` | Runtime lépések kétlépcsős teszt táblázata | #5: ACF smoke test tábla |

### Jövőbeli terv

Amikor a Spektra elér egy stabilabb fázist (Phase 8+), a smoke test
**automatizálható**:

- **GitHub Actions**: push után `curl --fail` az endpointra
- **PowerShell script**: `scripts/smoke-test.ps1` — egylépéses futtatás
- **Pre-push hook**: git push előtt automatikus ellenőrzés

Jelenleg ez nem prioritás, mert a rendszer még aktív fejlesztés alatt áll, és az agent
minden lépésnél úgyis lefuttatja.

---

## Összefoglaló

| Kérdés | Válasz |
|---|---|
| Mi az a smoke test? | A legegyszerűbb ellenőrzés: működik-e egyáltalán? |
| Mire jó? | Gyors sanity check minden változtatás után |
| Mire NEM jó? | Funkcionális, biztonsági, teljesítmény tesztelésre |
| A Spektrában hogyan működik? | Endpoint + debug.log ellenőrzés (kétlépcsős) |
| Ki futtatja? | Az AI agent, manuálisan, minden roadmap lépés után |
| Hová van regisztrálva? | Sehová — nincs CI, nincs hook, nincs script |
| Hol van az eredmény? | `bootstrap-infra-log.md` és `runtime-log.md` |
| Miért fontos a Spektrának? | Junction + placeholder válaszok elfedik a hibákat |
