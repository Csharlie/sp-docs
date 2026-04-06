# Spektra — WordPress Integration Explained (Közérthető + Senior Developer Szint)

# Mi történt valójában? (Egyszerű magyarázat)

A rendszer most eljutott oda, hogy:

**A frontend már WordPressből is tud adatot olvasni.**

Korábban:

```
Frontend → statikus mock adat
```

Most:

```
WordPress → Frontend
```

Ez azt jelenti, hogy ha WordPressben átírsz valamit, az a frontendben is megjelenhet.

Viszont amikor ezt összeraktuk, két probléma derült ki:

## Probléma #1 — Képadatok egy része elveszett

A WordPress többféle képadatot tud adni:

- nagy kép
- kisebb kép
- thumbnail
- responsive változatok

A mapper ezekből egy részt **eldobott**.

Tehát:

```
WP → több adat
mapper → kevesebb adat
frontend → hiányos adat
```

Ez nem törte el a rendszert, de **architekturálisan hibás**.

Senior szemmel:

Ez **contract fidelity hiba**.

---

## Probléma #2 — Üres blokkok megjelentek

Ha WordPressben egy blokk hiányosan volt kitöltve:

A rendszer:

- nem omlott össze
- de üres blokkot renderelt

Példák:

- üres gallery
- üres team
- üres stats

Ez rossz UX és elrejti a backend hibát.

Senior szemmel:

Ez **fail-soft policy drift**.

---

# Mostani architektúra

## Teljes rendszer

```
WordPress
    ↓
sp-infra
    ↓
REST API
    ↓
sp-benettcar mapper
    ↓
sp-benettcar normalizer
    ↓
spektra validator
    ↓
React UI
```

---

# Repo szintű architektúra

```
+-------------------+      +-------------------+      +-------------------+
|     sp-infra      |      |      spektra      |      |    sp-benettcar   |
|   (Infra layer)   |      |  (Platform layer) |      |   (Client layer)  |
+-------------------+      +-------------------+      +-------------------+
```

---

# sp-infra — WordPress oldal

Feladata:

- WordPress adat kiolvasás
- ACF mezők
- section assembly
- REST endpoint

```
sp-infra
├─ plugin/
├─ acf/
├─ response-builder/
└─ REST endpoint
```

---

# spektra — Platform

Feladata:

- SiteData típus
- Media típus
- Adapter contract
- Validator

```
spektra
├─ types/
├─ data/
└─ validation/
```

---

# sp-benettcar — Client

Feladata:

- adatforrás kiválasztás
- mapping
- cleanup
- React render

```
sp-benettcar
├─ src/data/
│  ├─ create-adapter.ts
│  ├─ wp-mapper.ts
│  ├─ normalize-site-data.ts
│  └─ site.ts
```

---

# Data flow részletesen

```
WordPress admin
    ↓
ACF mezők
    ↓
Response Builder
    ↓
REST endpoint
    ↓
create-adapter.ts
    ↓
mapWordPressSiteData()
    ↓
normalizeSiteData()
    ↓
validateSiteData()
    ↓
React render
```

---

# JSON mód vs WordPress mód

## JSON mód

```
VITE_DATA_SOURCE=json
```

Mock adat.

---

## WordPress mód

```
VITE_DATA_SOURCE=wordpress
```

Valós CMS adat.

---

# Mi történik ha WP-ben módosítasz

```
WP mező módosul
    ↓
REST endpoint
    ↓
Frontend lekéri
    ↓
Mapper
    ↓
Normalizer
    ↓
Validator
    ↓
React render
```

---

# Layer felelősségek

## Infra

Adatot ad

## Mapper

Fordít

## Normalizer

Tisztít

## Validator

Ellenőriz

## React

Renderel

---

# Miért fontos ez?

Ez már platform architektúra.

Nem:

- mock-first demo

Hanem:

- CMS driven platform

---

# Mostani állapot

## Kész

- WP endpoint
- mapper
- normalizer
- adapter
- validation

## Finomhangolás

- Media variants fix
- section skip policy

---

# Egyszerű összefoglaló

Most már:

**WP → frontend content pipeline működik**

A hátralévő munka:

- stabilitás
- adatminőség

---

# Egy mondatos verzió

Most már a WordPress a tartalomforrás, a frontend abból renderel.

---

# Senior verzió

The Spektra content pipeline is now assembled end-to-end. Remaining work focuses on boundary fidelity and render-safety hardening.

---

# Vizuális egyszerű diagram

```
WP
 ↓
Infra
 ↓
Mapper
 ↓
Normalizer
 ↓
Validator
 ↓
React
```

---

# Ez a Spektra CMS architektúra alapja

Ez az egyik legfontosabb mérföldkő a platformban.

---

# END

