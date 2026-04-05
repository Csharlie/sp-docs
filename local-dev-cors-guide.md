# Spektra Local Dev + CORS Guide

> Célközönség: junior fejlesztők, senior fejlesztők, külső kontribútorok.
> Utolsó frissítés: 2026-04-05
> **Implementációs státusz: Phase 5 complete** — CORS handler kész, tesztelve.

---

## Tartalomjegyzék

1. [Mi a Local Development a Spektrában](#mi-a-local-development-a-spektrában)
2. [Mi az a CORS](#mi-az-a-cors)
3. [Spektra Local Dev Architecture](#spektra-local-dev-architecture)
4. [allowed_origins config](#allowed_origins-config)
5. [Preflight request](#preflight-request)
6. [CORS headerek](#cors-headerek)
7. [WP default CORS felülírás](#wp-default-cors-felülírás)
8. [Debugging](#debugging)
9. [curl teszt](#curl-teszt)
10. [Hogyan adj hozzá új frontend origin-t](#hogyan-adj-hozzá-új-frontend-origin-t)
11. [Design Principles](#design-principles)
12. [Common Errors](#common-errors)
13. [Future Considerations](#future-considerations)
14. [Összefoglaló](#összefoglaló)

---

## Mi a Local Development a Spektrában

A Spektra **két külön alkalmazásból** áll, amik egymástól függetlenül futnak:

| Alkalmazás | Technológia | URL | Port |
|---|---|---|---|
| Frontend | Vite + React | `http://localhost:5174` | 5174 |
| WordPress | Apache (WAMP) + PHP | `http://benettcar.local` | 80 |

A két alkalmazás:
- **Külön processzben** fut (Vite dev server vs. Apache)
- **Külön porton** hallgat (5174 vs. 80)
- **Külön domain** alatt érhető el (`localhost` vs. `benettcar.local`)
- **Nincs közös memória**, nincs közös fájlrendszer

A frontend a WordPress-ből kéri le a tartalmat:

```
http://localhost:5174  →  fetch("http://benettcar.local/wp-json/spektra/v1/site")
```

Ez **cross-origin kérés**: a forrás origin (`localhost:5174`) és a cél origin
(`benettcar.local:80`) nem egyezik. A böngésző alapértelmezetten **blokkolja**.

Ezért kell CORS.

---

## Mi az a CORS

### Egyszerű magyarázat

CORS = **Cross-Origin Resource Sharing** (kereszt-eredet erőforrás-megosztás).

A böngésző biztonsági szabálya (**Same-Origin Policy**) megakadályozza, hogy egy weboldal
JavaScript-je más domain-ről kérjen adatot. A CORS egy **feloldó mechanizmus**: a szerver
HTTP headerekkel jelzi, hogy egy adott origin hozzáférhet.

### Miért létezik

Képzeld el, hogy egy rosszindulatú weboldal (`evil.com`) JavaScript-tel lekéri a
banki adataidat a `bank.hu`-ról — a Same-Origin Policy ezt akadályozza meg.

De a Spektra esetében a `localhost:5174` **nem rosszindulatú** — ez a saját frontendünk.
A CORS headerekkel mondjuk meg a böngészőnek: „ez az origin rendben van, engedd át."

### Mit lát a fejlesztő CORS nélkül

Ha a CORS nincs beállítva, a böngésző konzolban:

```
Access to fetch at 'http://benettcar.local/wp-json/spektra/v1/site'
from origin 'http://localhost:5174' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

A kérés **eljut a szerverre**, a szerver **visszaküld egy választ**, de a böngésző
**nem adja át a JavaScript-nek**. A Network tab-ban látható a 200-as válasz — de a
kód nem fér hozzá.

---

## Spektra Local Dev Architecture

```
┌──────────────────────────────────────┐
│  Frontend (Vite dev server)          │
│  http://localhost:5174               │
│  fetch("/wp-json/spektra/v1/site")   │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Browser                             │
│  "Ez cross-origin kérés."            │
│  "Van CORS engedély?"                │
│  [preflight?] OPTIONS → szerver      │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Apache (WAMP) :80                   │
│  benettcar.local vhost               │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  WordPress REST API                  │
│  /wp-json/spektra/v1/site            │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Spektra plugin                      │
│  ├── class-cors.php (priority 100)   │
│  ├── class-rest-controller.php       │
│  └── class-response-builder.php      │
└──────────────────┬───────────────────┘
                   │
                   ▼
          JSON válasz + CORS headerek
                   │
                   ▼
          Browser: "CORS OK" → React megkapja
```

A lényeg: a **CORS kezelés a plugin-ban történik**, nem az Apache-ban, nem a
Vite-ban, nem a WordPress core-ban. A `class-cors.php` felelős azért, hogy az
engedélyezett origin-ek hozzáférjenek.

---

## allowed_origins config

### Hol van

```
sp-benettcar/infra/config.php
```

Ez a fájl a WordPress runtime-ban a `spektra-config/config.php` Junction-ön keresztül
érhető el.

### Tartalom

```php
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

### Az `allowed_origins` szabályai

- **Whitelist** — csak a felsorolt origin-ek kapnak CORS headert
- **Nincs wildcard** — `*` soha nem használható (OWASP: Misconfigured CORS)
- **Kliens-specifikus** — minden kliens repo a saját origin listáját tartalmazza
- **Protokoll + host + port** kell — `http://localhost:5174` ≠ `https://localhost:5174`

### Mi számít origin-nek

Az origin a URL három részéből áll:

```
http://localhost:5174
└──┘   └───────┘ └──┘
proto   host     port
```

Ha bármelyik eltér → **más origin** → CORS kell.

| Eset | Ugyanaz az origin? |
|---|---|
| `http://localhost:5174` vs `http://localhost:5174` | ✅ Igen |
| `http://localhost:5174` vs `http://localhost:5175` | ❌ Nem (port) |
| `http://localhost:5174` vs `https://localhost:5174` | ❌ Nem (protokoll) |
| `http://localhost:5174` vs `http://benettcar.local` | ❌ Nem (host + port) |

---

## Preflight request

### Mi az

Bizonyos kérések előtt a böngésző **előzetes egyeztetést** kér a szervertől. Ezt
**preflight**-nak hívják — egy OPTIONS kérés, ami megkérdezi: „szabad lesz ezt csinálni?"

### Mikor történik

A böngésző preflight-ot küld, ha a kérés:
- Nem `GET`/`HEAD`/`POST`, vagy
- Van custom header (pl. `Authorization`, `Content-Type: application/json`), vagy
- A `Content-Type` nem `text/plain`, `multipart/form-data`, vagy `application/x-www-form-urlencoded`

Egy egyszerű `GET` (custom headerek nélkül) **nem** triggerel preflight-ot.

### Hogyan néz ki

```
OPTIONS /wp-json/spektra/v1/site HTTP/1.1
Host: benettcar.local
Origin: http://localhost:5174
Access-Control-Request-Method: GET
Access-Control-Request-Headers: Authorization
```

### Spektra válasz

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:5174
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
Vary: Origin
```

| Header | Jelentés |
|---|---|
| `204 No Content` | Sikeres, nincs body |
| `Allow-Methods` | Ezek a HTTP metódusok engedélyezettek |
| `Allow-Headers` | Ezek a custom headerek küldhetők |
| `Max-Age: 86400` | A böngésző 24 órán át cache-eli a preflight eredményt |

A preflight OK után a böngésző elküldi a **valódi GET kérést**.

---

## CORS headerek

A Spektra CORS handler (`class-cors.php`) az alábbi headereket kezeli:

### Minden válaszban (engedélyezett origin)

| Header | Érték | Cél |
|---|---|---|
| `Access-Control-Allow-Origin` | `http://localhost:5174` | Az engedélyezett origin (nem `*`) |
| `Vary: Origin` | — | Jelzi a cache-nek, hogy a válasz origin-függő |

### Preflight válaszban (OPTIONS)

| Header | Érték | Cél |
|---|---|---|
| `Access-Control-Allow-Methods` | `GET, OPTIONS` | Engedélyezett metódusok |
| `Access-Control-Allow-Headers` | `Content-Type, Authorization` | Engedélyezett custom headerek |
| `Access-Control-Max-Age` | `86400` | Preflight cache idő (másodperc) |

### Nem engedélyezett origin

| Akció | Cél |
|---|---|
| `header_remove('Access-Control-Allow-Origin')` | WP default CORS eltávolítása |
| `header_remove('Access-Control-Allow-Methods')` | WP default eltávolítása |
| `header_remove('Access-Control-Allow-Credentials')` | WP default eltávolítása |
| `Vary: Origin` | Cache-nek jelzés |

Eredmény: a böngésző **nem kap Allow-Origin headert** → blokkolja a választ.

### Miért fontos a `Vary: Origin`

Ha egy CDN vagy proxy cache-eli a választ, a `Vary: Origin` jelzi, hogy **az origin
alapján különböző válaszokat** kell cache-elni. Nélküle egy cache-elt válasz rossz
origin-nel válaszolhat.

---

## WP default CORS felülírás

### A probléma

A WordPress core tartalmaz egy beépített CORS kezelőt: `rest_send_cors_headers()`.
Ez a `rest_pre_serve_request` filter-en fut, **priority 10**-en.

A WordPress default viselkedése:

```php
header( 'Access-Control-Allow-Origin: ' . $_SERVER['HTTP_ORIGIN'] );
```

Ez **minden origin-t engedélyez** — bármely weboldal JavaScript-je hozzáférhet a
WordPress REST API-hoz. Ez biztonsági szempontból elfogadhatatlan.

### A Spektra megoldása

A `class-cors.php` **priority 100**-on fut — a WordPress default CORS (priority 10)
**után**. Két esetet kezel:

**Engedélyezett origin:**
```
WP default (priority 10): Access-Control-Allow-Origin: http://localhost:5174  ← OK
Spektra (priority 100):   Access-Control-Allow-Origin: http://localhost:5174  ← felülírja, ugyanaz
```

**Nem engedélyezett origin:**
```
WP default (priority 10): Access-Control-Allow-Origin: http://evil.com  ← ROSSZ
Spektra (priority 100):   header_remove('Access-Control-Allow-Origin')  ← eltávolítja
```

### Scope

A Spektra CORS filter **csak a `/spektra/` route-okra** vonatkozik:

```php
$route = $request->get_route();
if ( strpos( $route, '/spektra/' ) !== 0 ) {
    return $served; // nem Spektra route → nem nyúlunk hozzá
}
```

Más WP REST endpoint-ok (`/wp/v2/posts`, `/wp/v2/pages`, stb.) megtartják a WordPress
default CORS viselkedését. A Spektra nem módosítja a core működését a saját route-jain kívül.

---

## Debugging

### Browser DevTools

1. Nyisd meg a **Network** tabot
2. Keresd meg a `site` kérést (vagy `site?preview=true`)
3. Kattints rá → **Headers** tab

**Amit látnod kell (sikeres kérés):**

```
Response Headers:
  Access-Control-Allow-Origin: http://localhost:5174
  Vary: Origin
  X-Spektra-Version: 0.1.0
  Content-Type: application/json; charset=UTF-8
```

**Amit NEM szabad látnod:**

```
Access-Control-Allow-Origin: *
```

Ha `*`-ot látsz, a Spektra CORS filter nem fut — a WordPress default válaszol.

### Preflight ellenőrzés

A Network tab-ban keresd az **OPTIONS** kérést (a GET előtt jelenik meg):
- Status: **204**
- `Access-Control-Allow-Methods: GET, OPTIONS`
- `Access-Control-Allow-Headers: Content-Type, Authorization`

### Console errors

| Hibaüzenet | Jelentés |
|---|---|
| `No 'Access-Control-Allow-Origin' header` | Az origin nincs az `allowed_origins`-ban |
| `Response to preflight request doesn't pass` | OPTIONS válasz hibás |
| `The value of the 'Access-Control-Allow-Origin' header must not be the wildcard '*'` | WP default CORS válaszol, Spektra filter nem fut |

---

## curl teszt

A `curl` nem böngésző — nem érvényesíti a CORS-t. Viszont **a szerver viselkedését
tesztelhetjük** vele: a CORS headerek megjelennek-e a válaszban?

### Engedélyezett origin

```bash
curl -s -D - -o /dev/null \
  -H "Origin: http://localhost:5174" \
  http://benettcar.local/wp-json/spektra/v1/site
```

**Elvárt válasz headerek:**

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:5174
Vary: Origin
X-Spektra-Version: 0.1.0
```

### Nem engedélyezett origin

```bash
curl -s -D - -o /dev/null \
  -H "Origin: http://evil.com" \
  http://benettcar.local/wp-json/spektra/v1/site
```

**Elvárt:** nincs `Access-Control-Allow-Origin` header a válaszban (a `header_remove()`
eltávolította).

### Preflight

```bash
curl -s -D - -o /dev/null \
  -X OPTIONS \
  -H "Origin: http://localhost:5174" \
  -H "Access-Control-Request-Method: GET" \
  http://benettcar.local/wp-json/spektra/v1/site
```

**Elvárt:**

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: http://localhost:5174
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
Vary: Origin
```

### Origin nélkül (same-origin vagy curl)

```bash
curl -s -D - -o /dev/null \
  http://benettcar.local/wp-json/spektra/v1/site
```

**Elvárt:** 200 OK, nincs `Access-Control-Allow-Origin` header (nem cross-origin kérés).

---

## Hogyan adj hozzá új frontend origin-t

### Lépések

**1. Config módosítás**

Szerkeszd a kliens config fájlt:

```
sp-benettcar/infra/config.php
```

Add hozzá az új origin-t:

```php
'allowed_origins' => [
    'http://localhost:5174',       // Vite dev server
    'http://localhost:3000',       // új frontend origin
],
```

**2. WordPress újratöltés**

A config betöltése minden kérésnél megtörténik (nincs persistent cache), de az
Apache-ot érdemes újraindítani az opcache miatt:

```bash
# WAMP: restart Apache
```

**3. Teszt**

```bash
curl -s -D - -o /dev/null \
  -H "Origin: http://localhost:3000" \
  http://benettcar.local/wp-json/spektra/v1/site
```

Ellenőrizd, hogy az `Access-Control-Allow-Origin: http://localhost:3000` megjelenik.

### Fontos szabályok

- **Pontos origin** kell — `http://localhost:3000` ≠ `http://localhost:3000/`
- **Protokoll számít** — `http` ≠ `https`
- **Port számít** — ha nincs port, az alapértelmezett (80/443) érvényes
- **Trailing slash tilos** — `http://localhost:3000/` nem valid origin

---

## Design Principles

A Spektra CORS kezelés 5 tervezési elvre épül:

### 1. Whitelist Only

Csak az explicit felsorolt origin-ek kapnak hozzáférést. Nincs pattern matching,
nincs regex, nincs subdomain wildcard.

**Miért:** egyértelmű, auditálható, nem lehet véletlenül túl sokat engedni.

### 2. No Wildcard

`Access-Control-Allow-Origin: *` **soha** nem használható. Minden válaszban a
konkrét origin string szerepel.

**Miért:** a `*` megengedi, hogy bármely weboldal JavaScript-je lekérje az adatokat.
OWASP Top 10: Security Misconfiguration.

### 3. Client Specific

Minden kliens repo a **saját** `allowed_origins` listáját tartalmazza. A plugin
forráskód nem tartalmaz hardcoded origin-eket.

**Miért:** ha egy új kliens jön, nem a plugin kódjába kell nyúlni — a kliens overlay
config-ja tartalmazza az origin listát.

### 4. Secure Default

Ha nincs origin header a kérésben → nincs CORS válasz. Ha az origin nincs a
listában → `header_remove()`. Az alapértelmezés **mindig a blokkolás**.

**Miért:** fail-closed > fail-open. Biztonsági beállításoknál az alapértelmezés
mindig a szigorúbb legyen.

### 5. Minimal Headers

Csak a szükséges CORS headerek kerülnek a válaszba. Nincs `Allow-Credentials`
(nincs cookie-alapú auth), nincs `Expose-Headers` (nincs custom válasz header olvasás).

**Miért:** a minimális felszín csökkenti a támadási vektort.

---

## Common Errors

### „Missing origin" — nincs `Access-Control-Allow-Origin`

**Tünet:** a böngésző blokkolja a kérést, a Network tab-ban nincs `Allow-Origin` header.

**Okok:**
- Az origin nincs az `allowed_origins` listában
- Az origin formátuma hibás (pl. trailing slash, rossz port)
- A plugin nem aktív
- A config.php nem töltődik be (Junction hibás)

**Megoldás:** ellenőrizd a config.php `allowed_origins` tömböt és a curl teszttel validáld.

### Wrong port

**Tünet:** `http://localhost:5173`-ról jön a kérés, de az `allowed_origins`-ban
`http://localhost:5174` van.

**Ok:** a Vite más portra indult (pl. az 5174 foglalt volt).

**Megoldás:** ellenőrizd a Vite dev server tényleges portját, frissítsd a config-ot.

### Cache issue

**Tünet:** módosítottad az `allowed_origins`-t, de a változás nem érvényesül.

**Okok:**
- PHP opcache — Apache restart kell
- Böngésző preflight cache — `Access-Control-Max-Age: 86400` (24 óra)
- CDN / proxy cache (production-ben)

**Megoldás:**
- Apache restart vagy opcache invalidálás
- Böngészőben: hard refresh (Ctrl+Shift+R) vagy Incognito ablak
- Vagy: curl-lel tesztelj (nincs preflight cache)

### Preflight fail

**Tünet:** az OPTIONS kérés 200-at ad vissza 204 helyett, vagy hiányzik az
`Allow-Methods` header.

**Okok:**
- A plugin CORS filter nem fut (plugin inaktív, hook hiba)
- Más plugin vagy .htaccess felülírja az OPTIONS viselkedést

**Megoldás:** ellenőrizd a plugin aktiválást (`wp plugin list`), és teszteld curl-lel.

### Mixed content (production)

**Tünet:** `https://` frontend `http://` backend-et hív — böngésző blokkolja.

**Ok:** ez nem CORS hiba, hanem **mixed content** blokkolás. HTTPS → HTTP kérés tilos.

**Megoldás:** a WordPress-t is HTTPS-en futtasd.

---

## Future Considerations

### Production domain

Production-ben az `allowed_origins` a kliens végleges domain-jét tartalmazza:

```php
'allowed_origins' => [
    'https://benettcar.hu',
    'https://www.benettcar.hu',
],
```

Fontos: **HTTPS** — production-ben plain HTTP nem elfogadható.

### Multi-client

Ha több kliens fut egy WordPress-en (különböző vhost-ok), minden kliensnek saját
config overlay-e van saját `allowed_origins` listával. A plugin a betöltött config
alapján dönt — nem globális beállítás.

### Staging environment

Staging környezetben az `allowed_origins` a staging domain-jét tartalmazza:

```php
'allowed_origins' => [
    'https://staging.benettcar.hu',
    'http://localhost:5174',  // dev is mehet, ha kell
],
```

### Credentials (jövőbeli)

Ha a jövőben autentikált kérések kellenek (pl. preview mode cookie-val), az
`Access-Control-Allow-Credentials: true` headert is be kell állítani. Ez **nem
kompatibilis** a `*` wildcard-dal — ami újabb ok, hogy a Spektra explicit origin
listát használ.

---

## Összefoglaló

| Kérdés | Válasz |
|---|---|
| Miért kell CORS? | A frontend és backend különböző origin-en fut |
| Ki kezeli? | `class-cors.php`, priority 100 |
| Honnan jön a whitelist? | `sp-benettcar/infra/config.php` → `allowed_origins` |
| Van wildcard? | Nem, soha |
| Mi történik ismeretlen origin-nel? | `header_remove()` → böngésző blokkolja |
| Mi a preflight? | OPTIONS → 204, Allow-Methods, Allow-Headers, Max-Age |
| Felülírja a WP default-ot? | Igen, priority 100 vs. WP priority 10 |
| Hogyan teszteld? | curl + Origin header, vagy Browser DevTools → Network |
| Hogyan adj hozzá új origin-t? | `config.php` → `allowed_origins` → Apache restart → teszt |
| Mi a `Vary: Origin`? | Jelzi a cache-nek, hogy a válasz origin-függő |
