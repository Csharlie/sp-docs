# CORS — Mi ez, miért kell, hogyan működik a Spektrában

> Célközönség: junior fejlesztők, senior fejlesztők, külső kontribútorok.
> Utolsó frissítés: 2026-04-05

---

## Tartalomjegyzék

1. [Mi az a CORS?](#mi-az-a-cors)
2. [Miért létezik?](#miért-létezik)
3. [Hogyan működik?](#hogyan-működik)
4. [A Spektra CORS problémája](#a-spektra-cors-problémája)
5. [A megoldás: class-cors.php](#a-megoldás-class-corsphp)
6. [A döntési fa](#a-döntési-fa)
7. [Konkrét Spektra példák](#konkrét-spektra-példák)
8. [A WordPress CORS probléma](#a-wordpress-cors-probléma)
9. [Gyakori CORS hibák és megoldásuk](#gyakori-cors-hibák-és-megoldásuk)
10. [Vary: Origin — miért fontos](#vary-origin--miért-fontos)
11. [Biztonsági megfontolások](#biztonsági-megfontolások)
12. [Összefoglaló](#összefoglaló)

---

## Mi az a CORS?

CORS = **Cross-Origin Resource Sharing** (kereszt-eredet erőforrás-megosztás).

Egy HTTP header-alapú mechanizmus, amivel a szerver **engedélyt ad** a böngészőnek,
hogy egy másik domain/port/protokoll felől érkező JavaScript kérést átengedjen.

### A legegyszerűbb magyarázat

Te vagy a portás egy épületben (böngésző). Valaki bejön és azt mondja: „A 3-as
szobából jövök, szeretném az 5-ös szoba aktáit." Te felhívod az 5-ös szobát
(szerver), és megkérdezed: „Adhatok aktát a 3-asnak?"

- Ha az 5-ös azt mondja „igen" → átadod az aktát
- Ha az 5-ös azt mondja „nem" vagy nem válaszol → nem adod ki

A CORS **nem a szerver védi magát** (a szerver amúgy is feldolgozza a kérést).
A CORS a **böngésző biztonsági mechanizmusa** — ő dönt arról, hogy a JavaScript
hozzáférhet-e a válaszhoz.

---

## Miért létezik?

### A Same-Origin Policy

A böngészők alapvető biztonsági szabálya: a **Same-Origin Policy** (azonos eredet
szabály). Ez azt mondja:

> Egy weboldal JavaScript-je csak **saját origin-jéről** kérhet le adatot.

Mi az origin? Három dolog együtt:

```
http://localhost:5174
└─┬──┘ └────┬────┘└─┬┘
protokoll  host   port
```

Ha bármelyik **egy is** különbözik → **más origin** → **blokkolva**.

| Kérés innen | Kérés ide | Azonos origin? |
|---|---|---|
| `http://localhost:5174` | `http://localhost:5174/api` | ✅ Igen |
| `http://localhost:5174` | `http://localhost:3000` | ❌ Nem (port) |
| `http://localhost:5174` | `http://benettcar.local` | ❌ Nem (host + port) |
| `http://example.com` | `https://example.com` | ❌ Nem (protokoll) |

### Miért jó ez?

Képzeld el, hogy bejelentkezel a bankodba (`bank.com`). Aztán megnyitod egy másik
tabon a `rosszoldal.com`-ot. Ha nincs Same-Origin Policy, a rossz oldal JavaScript-je
simán lekérhetné a `bank.com/api/account-balance`-t — mert a böngésződ a bank
session cookie-ját automatikusan elküldi.

A Same-Origin Policy megakadályozza ezt: `rosszoldal.com` JavaScript-je **nem
olvashatja el** a `bank.com`-ról kapott választ.

### De mi van, ha jogos a kérés?

Pont ez a CORS lényege. A Spektra frontend (`localhost:5174`) és a backend
(`benettcar.local:80`) jogosan kommunikálnak — de a böngésző nem tudja
megkülönböztetni a jogos cross-origin kérést a rosszindulatútól.

A CORS-szal a **szerver jelzi**: „Ez az origin OK, engedjétek."

---

## Hogyan működik?

Két fázis van: **preflight** (előzetes egyeztetés) és a **tényleges kérés**.

### Egyszerű kérés (simple request)

Ha a kérés „egyszerű" (GET, nincs speciális header), a böngésző **azonnal elküldi**,
de a visszakapott választ csak akkor adja át a JavaScript-nek, ha van benne CORS
header:

```
Böngésző (JS)                        Szerver
     │                                  │
     ├─── GET /spektra/v1/site ────────►│
     │    Origin: http://localhost:5174  │
     │                                  │
     │◄── 200 OK ──────────────────────┤
     │    Access-Control-Allow-Origin:  │
     │    http://localhost:5174         │
     │                                  │
     │  "Allow-Origin megegyezik?       │
     │   Igen → JS megkapja a választ"  │
```

### Preflight kérés (az „előzetes egyeztetés")

Ha a kérésnek van speciális header-je (pl. `Authorization`, `Content-Type: application/json`),
a böngésző **két lépésben** dolgozik:

```
Böngésző (JS)                        Szerver
     │                                  │
     │  "Van Authorization header,      │
     │   előbb megkérdezem..."          │
     │                                  │
     ├─── OPTIONS /spektra/v1/site ────►│    ← preflight kérés
     │    Origin: http://localhost:5174  │
     │    Access-Control-Request-Method: │
     │      GET                         │
     │    Access-Control-Request-Headers:│
     │      Authorization               │
     │                                  │
     │◄── 204 No Content ──────────────┤    ← preflight válasz
     │    Access-Control-Allow-Origin:  │
     │      http://localhost:5174       │
     │    Access-Control-Allow-Methods: │
     │      GET, OPTIONS                │
     │    Access-Control-Allow-Headers: │
     │      Content-Type, Authorization │
     │    Access-Control-Max-Age: 86400 │
     │                                  │
     │  "Rendben, engedélyezve."        │
     │                                  │
     ├─── GET /spektra/v1/site ────────►│    ← tényleges kérés
     │    Origin: http://localhost:5174  │
     │    Authorization: Bearer ...     │
     │                                  │
     │◄── 200 OK ──────────────────────┤    ← tényleges válasz
     │    Access-Control-Allow-Origin:  │
     │      http://localhost:5174       │
     │    { "site": {...}, ... }        │
```

### A headerek összefoglalója

| Header | Ki küldi | Mikor | Mit jelent |
|---|---|---|---|
| `Origin` | Böngésző → szerver | Minden cross-origin kérés | „Innen jövök" |
| `Access-Control-Allow-Origin` | Szerver → böngésző | Válaszban | „Ez az origin OK" |
| `Access-Control-Allow-Methods` | Szerver → böngésző | Preflight válaszban | „Ezek a metódusok OK" |
| `Access-Control-Allow-Headers` | Szerver → böngésző | Preflight válaszban | „Ezek a headerek OK" |
| `Access-Control-Max-Age` | Szerver → böngésző | Preflight válaszban | „Ennyi mp-ig ne kérdezz újra" |
| `Vary: Origin` | Szerver → böngésző | Válaszban | „A válasz az origintől függ" |

---

## A Spektra CORS problémája

A Spektra két alkalmazásból áll, két origin-en:

```
Frontend (React + Vite)          Backend (WordPress + API)
http://localhost:5174             http://benettcar.local
     │                                  │
     │  fetch('/wp-json/spektra/v1/site')
     │                                  │
     │  ❌ Origin mismatch!             │
     │  Böngésző: "CORS header kell"    │
```

**CORS nélkül** a Spektra frontend **nem tud adatot lekérni** a backend-ről.
A `fetch()` hívás eredménye: `TypeError: Failed to fetch` — a böngésző blokkolja.

### Miért nem ugyanazon az origin-en fut?

Mert ez headless architektúra:

| Komponens | Technológia | Port | Miért külön? |
|---|---|---|---|
| Frontend | Vite + React | 5174 | Hot reload, TypeScript, SPA |
| Backend | Apache + WordPress | 80 | PHP, MySQL, ACF, admin felület |

Egy origin-re tenni (pl. Apache reverse proxy) lehetséges, de:
- Bonyolítja a dev setupot
- Hot reload megszakadhat
- A produkciós környezet amúgy is különálló (CDN + API szerver)

CORS a **standard megoldás** erre a helyzetre.

---

## A megoldás: class-cors.php

A Spektra a CORS kezelést egyetlen PHP osztályban valósítja meg:
`sp-infra/plugin/spektra-api/includes/class-cors.php`

### Amit csinál

1. A `rest_pre_serve_request` WordPress filteren keresztül **minden REST válasz előtt**
   lefut
2. Megvizsgálja a kérés **route-ját** — csak `/spektra/` route-okra hat
3. Megvizsgálja az **Origin** headert
4. Összeveti az **allowed_origins** listával (kliens configból)
5. Engedélyezett → CORS headerek, nem engedélyezett → headerek eltávolítása

### Amit NEM csinál

- Nem hat a WordPress egyéb REST route-jaira (`/wp/v2/posts`, stb.)
- Nem használ `*` (wildcard) origin-t
- Nem engedélyez credential-eket (cookie-k cross-origin — jelenleg nincs rá szükség)
- Nem módosítja a válasz body-t

---

## A döntési fa

Ezt a logikát követi a `add_cors_headers()` metódus:

```
Beérkező kérés
     │
     ├── Route /spektra/*?
     │   ├── NEM → return (nem nyúlunk hozzá)
     │   └── IGEN ↓
     │
     ├── Van Origin header?
     │   ├── NEM → return (nem cross-origin, nem kell CORS)
     │   └── IGEN ↓
     │
     ├── Origin benne van az allowed_origins-ban?
     │   │
     │   ├── NEM → header_remove('Access-Control-Allow-Origin')
     │   │         header_remove('Access-Control-Allow-Methods')
     │   │         header_remove('Access-Control-Allow-Credentials')
     │   │         header('Vary: Origin')
     │   │         return (WP default CORS törölve)
     │   │
     │   └── IGEN ↓
     │
     ├── header('Access-Control-Allow-Origin: <origin>')
     ├── header('Vary: Origin')
     │
     └── OPTIONS (preflight)?
         ├── NEM → return (normál GET, headerek már beállítva)
         └── IGEN → Allow-Methods: GET, OPTIONS
                    Allow-Headers: Content-Type, Authorization
                    Max-Age: 86400
                    status: 204
                    $served = true (ne küldjön JSON body-t a WP)
```

---

## Konkrét Spektra példák

### 1. Normál működés — engedélyezett frontend

```
Frontend: http://localhost:5174
Config:   allowed_origins: ['http://localhost:5174']

GET /wp-json/spektra/v1/site
Origin: http://localhost:5174

← 200 OK
← Access-Control-Allow-Origin: http://localhost:5174
← Vary: Origin
← X-Spektra-Version: 0.1.0
← {"site":[],"navigation":[],"pages":[]}

→ Böngésző: "Origin egyezik, JS megkapja a választ." ✅
```

### 2. Támadó oldal — ismeretlen origin

```
Támadó: http://evil.com

GET /wp-json/spektra/v1/site
Origin: http://evil.com

← 200 OK
← (NINCS Access-Control-Allow-Origin header)
← Vary: Origin

→ Böngésző: "Nincs CORS engedély, JS nem kapja meg a választ." ❌
```

A szerver maga a választ **amúgy is elküldte** (200 OK) — a CORS nem szerver-oldali
védelem. De a támadó oldal JavaScript-je **nem tudja elolvasni** a választ, mert a
böngésző blokkolja.

### 3. Preflight — speciális headerrel

```
Frontend: http://localhost:5174

OPTIONS /wp-json/spektra/v1/site
Origin: http://localhost:5174
Access-Control-Request-Method: GET
Access-Control-Request-Headers: Authorization

← 204 No Content
← Access-Control-Allow-Origin: http://localhost:5174
← Access-Control-Allow-Methods: GET, OPTIONS
← Access-Control-Allow-Headers: Content-Type, Authorization
← Access-Control-Max-Age: 86400

→ Böngésző: "24 óráig nem kell újra preflight-olni." ✅
→ Böngésző: elküldi a tényleges GET kérést.
```

### 4. Nem-Spektra route — nem bántjuk

```
GET /wp-json/wp/v2/types
Origin: http://barmelyik.com

← 200 OK
← Access-Control-Allow-Origin: http://barmelyik.com  (WP default!)

→ Mi nem szóltunk bele — nem /spektra/ route.
```

---

## A WordPress CORS probléma

### Mit csinál a WP alapból?

A WordPress `rest_send_cors_headers()` függvénye (priority 10) **minden origin-t
átereszt**:

```php
// WordPress core: wp-includes/rest-api.php
header( 'Access-Control-Allow-Origin: ' . get_http_origin() );
```

Ez azt jelenti, hogy ha `evil.com` kéri a `/wp-json/spektra/v1/site` endpointot,
a WordPress alapból rámondja: „Persze, tessék!"

### Miért csinálja ezt a WP?

Mert a WordPress általános célú CMS. Nem tudja, hogy melyik origin jogos és melyik
nem — ezt a felhasználóra (a plugin-ra) bízza. A default viselkedés „nyitott",
hogy a headless/Gutenberg/JS felhasználási esetek működjenek.

### Hogyan írjuk felül?

A Spektra CORS filter **priority 100**-on fut (WP default: 10), tehát utána:

```
Priority  10: WP default → header('Allow-Origin: evil.com')
Priority 100: Spektra    → header_remove('Allow-Origin')  ← felülírás
```

Nem engedélyezett origin-nél **eltávolítjuk** a WP által beállított headert.
Engedélyezett origin-nél **felülírjuk** a sajátunkkal (technikailag ugyanaz, de
explicit).

Fontos: ez **csak a spektra route-okra** vonatkozik. A `/wp/v2/` route-okat nem
piszkáljuk — ott marad a WP default viselkedés.

---

## Gyakori CORS hibák és megoldásuk

### 1. „Access to fetch has been blocked by CORS policy"

```
Access to fetch at 'http://benettcar.local/wp-json/spektra/v1/site'
from origin 'http://localhost:5174' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

**Ok:** A szerver nem küldött `Access-Control-Allow-Origin` headert.

**Megoldás:** Ellenőrizd, hogy a `config.php`-ban az `allowed_origins` tartalmazza-e
az origin-t. Figyelj a **portra** is — `http://localhost:5174` és `http://localhost:5173`
más origin!

### 2. „CORS policy: Response to preflight request doesn't pass"

```
Access to fetch has been blocked by CORS policy:
Response to preflight request doesn't pass access control check.
```

**Ok:** Az OPTIONS kérésre nem érkezett helyes CORS válasz.

**Megoldás:** Ellenőrizd, hogy a preflight (OPTIONS) eléri-e a PHP-t. Apache/nginx
néha saját maga válaszol az OPTIONS-re, mielőtt a PHP futna.

### 3. „The value of 'Access-Control-Allow-Origin' header must not be the wildcard '*'"

**Ok:** Valaki `*`-ot állított be, de a kérés credential-eket is küld.

**Megoldás:** A Spektra nem használ `*`-ot, de ha mégis felmerülne — soha ne `*`,
mindig explicit lista.

### 4. A böngésző cache-eli a preflight-ot, és az „öreg" választ kapja

**Ok:** `Access-Control-Max-Age` túl magas volt, és a config változott.

**Megoldás:** Böngésző DevTools → Network → Disable cache, vagy hard refresh.
A Spektra `Max-Age: 86400` (24 óra) — ez általában rendben van dev módban. Ha baj
van, a böngésző cache törlésekor resetelődik.

---

## Vary: Origin — miért fontos

### A probléma

Tegyük fel, hogy nincs `Vary: Origin`, és van egy **cache proxy** (CDN, Varnish,
böngésző cache) a szerver előtt:

```
1. kérés (localhost:5174):
   ← Allow-Origin: http://localhost:5174
   ← (cache elmenti)

2. kérés (evil.com):
   ← Proxy: "Van cache, tessék!"
   ← Allow-Origin: http://localhost:5174  ← ROSSZ! evil.com kapta a 5174-es headert
```

### A megoldás

```
Vary: Origin
```

Ez azt mondja a cache-nek: „A válasz **függ az Origin headertől**. Különböző origin-ekre
**különböző cache bejegyzést** tarts."

```
1. kérés (localhost:5174):
   ← Allow-Origin: http://localhost:5174
   ← Vary: Origin
   ← (cache: origin=localhost:5174 → ez a válasz)

2. kérés (evil.com):
   ← Proxy: "Nincs cache ehhez az origin-hez, kérdezzük a szervert"
   ← (nincs Allow-Origin)
   ← Vary: Origin
   ← (cache: origin=evil.com → ez a válasz)
```

### Mikor kell?

**Mindig**, ha a CORS válasz origin-specifikus (nem `*`). A Spektra minden spektra
route-ra küld `Vary: Origin`-t — engedélyezett és nem engedélyezett origin-re is.

---

## Biztonsági megfontolások

### Miért nem `Access-Control-Allow-Origin: *`?

A `*` azt jelenti: **bármilyen weboldal JavaScript-je** lekérheti az adatokat.
Ha a Spektra API-n valaha érzékeny adat is lesz (preview tartalom, draft oldalak),
a `*` azt is kiadná bárkinek.

OWASP besorolás: **Security Misconfiguration** — a túl nyitott CORS policy
könnyíti a támadó dolgát.

### Mi van, ha a szerver a kérést amúgy is feldolgozza?

Igen, a CORS **nem szerver-oldali védelem**. Egy `curl` parancs vagy Postman simán
lekérheti bármilyen CORS header nélkül az adatot — mert ők nem böngészők, nem
érvényesítenek Same-Origin Policy-t.

A CORS pont a **böngészőben futó JavaScript-et** védi:
- `evil.com` JS-je → nem kapja meg a választ (CORS blokkol)
- `curl` a terminálból → megkapja (nincs CORS check)

Ha szerver-oldali védelmet akarunk (pl. csak autentikált kérések engedélyezése),
az **authentikáció** (API key, JWT, cookie) feladata, nem a CORS-é.

### Miért nem engedélyezünk credential-eket?

A Spektra API publikus (`permission_callback: __return_true`). Nincs szükség
cookie-k cross-origin küldésére. Ha lenne (pl. WP preview session), azt explicit
külön kellene kezelni, `Access-Control-Allow-Credentials: true` header-rel.

---

## Összefoglaló

| Kérdés | Válasz |
|---|---|
| Mi az a CORS? | Böngésző biztonsági mechanizmus — engedélyezi a cross-origin kéréseket |
| Miért kell a Spektrának? | Frontend (:5174) és backend (:80) más origin-en fut |
| Ki valósítja meg? | `class-cors.php` — a `rest_pre_serve_request` filteren |
| Honnan jön az engedélyezett lista? | `config.php` → `allowed_origins` tömb |
| Mi történik ismeretlen origin-nel? | `header_remove()` — nincs CORS header → böngésző blokkol |
| Mi a preflight? | OPTIONS kérés — „szabad-e ezt a kérést elküldeni?" |
| Miért nem `*`? | Biztonsági kockázat (OWASP Misconfiguration) |
| Mi a `Vary: Origin`? | Cache-nek jelzi, hogy a válasz origin-függő |
| A WP mit csinál alapból? | Minden origin-t átereszt — a Spektra felülírja priority 100-on |
| Ez szerver-oldali védelem? | Nem — a böngészőt védi, `curl`/Postman ellen nem segít |
