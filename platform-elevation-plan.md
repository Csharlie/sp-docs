# Spektra Platform Elevation Plan

> **Cél:** Azonosítani, mit kell platform szintre emelni, hogy új kliens létrehozása ne a benettcar másolásán alapuljon  
> **Dátum:** 2026-04-12  
> **Kontextus:** sp-benettcar és sp-exotica összehasonlítása — a két kliens felépítése feltárta az ismétlődő mintákat  
> **Státusz:** Javaslat (draft)

---

## Tartalomjegyzék

- [I. rész — Közérthető magyarázat](#i-rész--közérthető-magyarázat)
- [II. rész — Senior Architect magyarázat](#ii-rész--senior-architect-magyarázat)

---

# I. rész — Közérthető magyarázat

## Mi a probléma?

Amikor egy új ügyfélnek (pl. Exotica by Bella) készítünk weboldalt a Spektra rendszerben, a jelenlegi módszer az, hogy a meglévő ügyfél (Benettcar autókereskedés) fájljait **kimásoljuk** és átírjuk az új ügyfélre. Ez működik, de:

- Könnyű elfelejteni valamit (pl. a `tsx` és `@types/node` csomagot mi is elfelejtettük)
- A másolt fájlok idővel eltérnek egymástól (a benettcar-nál újabb ESLint verzió van, mint az exoticánál)
- Nem egyértelmű, mit KELL módosítani és mit NEM kell hozzányúlni
- Hibalehetőség: ha a platformban javítunk valamit, az nem jut el automatikusan a kliens másolatokba

## Mit kellene csinálni?

Három szintű megoldást javasolunk:

### 1. szint: Azonnali — „Ne másold, hanem használd"

Vannak fájlok, amelyek **100%-ban azonosak** a két kliensben. Ezeket ki kell vinni egy közös helyre, és onnan importálni. Így ha javítunk rajtuk, minden kliens azonnal megkapja.

Ilyen fájlok:
| Fájl | Mit csinál |
|------|-----------|
| `create-adapter.ts` | Eldönti, honnan jönnek az adatok (statikus / WordPress) |
| `wp-mapper.ts` | WordPress API válaszokat TypeScript-re fordítja |
| `export-seed.ts` | Kiszedi az adatokat a site.ts-ből |
| `asset-loader.mjs` | Képek kezelése Node-ban |
| `register-asset-hooks.mjs` | Asset loader regisztrálása |
| `tsconfig.json` (seed) | TypeScript beállítások a seed eszközökhöz |

### 2. szint: Középtávú — „Szórd szét a sablonból"

Vannak fájlok, amelyeknek a **szerkezete mindig ugyanaz**, de a tartalmuk az ügyféltől függ. Ezeknél a megoldás: egy konfigból generálni kell őket.

Például a PHP fájlok (ACF field group-ok) mindig így néznek ki:
```
csoport neve = "{prefix}_{szekció}"
mező neve = "{prefix}_{szekció}_{mező}"
típus = "text" / "textarea" / "image" / "repeater"
hely = front_page
```

Ahelyett, hogy kézzel írjuk mind a 6 PHP fájlt, egy egyszerű konfigból lehetne generálni őket.

### 3. szint: Hosszútávú — „A kliens scaffold legyen egy parancs"

A végcél: `spektra create-client exotica --prefix eb --sections hero,about,products,animals,gallery,contact` — és minden fájl legenerálódik a megfelelő tartalommal.

## Miért fontos?

| Szempont | Most (copy-paste) | Platform szint után |
|----------|-------------------|---------------------|
| Új kliens létrehozása | ~2 óra manuális másolás | ~10 perc parancssorból |
| Hiányzó dependency | Könnyen kimarad | Automatikusan benne van |
| Platform javítás eljut a klienshez? | Nem | Igen (shared csomagoknál) |
| Verzió drift | Garantált | Kontrollált |

---

# II. rész — Senior Architect magyarázat

## Analízis: Benettcar vs Exotica fájl-összehasonlítás

Az exotica felépítése során részletes összehasonlítás készült a benettcar-ral. A fájlok 4 kategóriába sorolhatók:

### Kategória A: 100% azonos — azonnali deduplikáció

| Fájl | Hol van most | Javasolt hely |
|------|-------------|---------------|
| `src/data/create-adapter.ts` | Mindkét kliensben | `sp-platform/packages/data/create-adapter.ts` |
| `src/data/wp-mapper.ts` | Mindkét kliensben | `sp-platform/packages/data/wp-mapper.ts` |
| `infra/seed/export-seed.ts` | Mindkét kliensben | `sp-platform/packages/seed/export-seed.ts` |
| `infra/seed/asset-loader.mjs` | Mindkét kliensben | `sp-platform/packages/seed/asset-loader.mjs` |
| `infra/seed/register-asset-hooks.mjs` | Mindkét kliensben | `sp-platform/packages/seed/register-asset-hooks.mjs` |
| `infra/seed/tsconfig.json` | Mindkét kliensben | `sp-platform/packages/seed/tsconfig.json` (base config, extend) |

**Különbségek:** Kizárólag `@package` JSDoc comment — semmi funkcionális eltérés.

**Kockázat:** Nulla. Ezek a fájlok nem tartalmaznak kliens-specifikus logikát.

**Implementáció:**
```
sp-platform/
  packages/
    data/
      create-adapter.ts    ← kliens importálja: import { createAdapter } from '@spektra/data'
      wp-mapper.ts         ← kliens importálja: import { mapWordPressResponse } from '@spektra/data'
    seed/
      export-seed.ts
      asset-loader.mjs
      register-asset-hooks.mjs
      tsconfig.seed.json   ← kliensek extends-elik
```

**Breaking change:** A kliens `import` path-ek módosulnak — de ez egy egypontos átírás + a package.json-ba kell egy `@spektra/data` workspace dependency.

---

### Kategória B: Azonos struktúra, kliens-specifikus tartalom — generálható

| Fájl | Mit tartalmaz kliens-specifikusan |
|------|----------------------------------|
| `infra/seed/mapping.ts` | SectionMapping[] — szekció típusok, ACF mező nevek, extract függvények |
| `src/data/normalize-site-data.ts` | switch-case szekció típusonként, normalize függvények szekciónként |
| `infra/acf/builders.php` | Builder függvények szekciónként, ACF field olvasás + JSON struktúra |
| `infra/acf/sections/*.php` | ACF field group definíciók — nyers PHP tömbök |

**Közös minta:** Mind a 4 fájltípusnál a szekció-lista + mező-lista a paraméter. A pattern fix, a data változó.

#### B1: mapping.ts generálás

**Jelenlegi állapot:**
```typescript
// Közös (azonos mindkét kliensben):
type FieldKind = 'scalar' | 'image' | 'repeater';
interface FieldMapping { acfKey: string; kind: FieldKind; extract: (data: unknown) => unknown; }
interface SectionMapping { sectionType: string; fields: FieldMapping[]; }
function str(val: unknown): string { ... }
function bool(val: unknown): string { ... }
function extractCta(data: Record<string, unknown>, prefix: string) { ... }
function extractMedia(data: Record<string, unknown>, key: string) { ... }

// Kliens-specifikus:
export const sectionMappings: SectionMapping[] = [
  { sectionType: 'eb-hero', fields: [ /* eb_ prefixed fields */ ] },
  { sectionType: 'eb-about', fields: [ /* eb_ prefixed fields */ ] },
  // ...
];
```

**Javaslat:** A típusok és helper-ek menjenek `@spektra/seed`-be. A kliens-specifikus mapping maradjon a kliensben, de egy rövidebb szintaxissal:

```typescript
// sp-exotica/infra/seed/mapping.ts (jövő)
import { defineMapping, SectionMapping } from '@spektra/seed';

export const sectionMappings: SectionMapping[] = [
  defineMapping('eb-hero', d => ({
    eb_hero_title: d.title,
    eb_hero_subtitle: d.subtitle,
    eb_hero_background_image: { kind: 'image', value: d.backgroundImage },
    eb_hero_primary_cta_text: d.primaryCTA?.text,
    eb_hero_primary_cta_href: d.primaryCTA?.href,
  })),
  // ...
];
```

Ez ~60%-kal csökkentené a mapping.ts méretét, miközben type-safe marad.

#### B2: normalize-site-data.ts generálás

**Jelenlegi állapot:**
```typescript
// Közös helper-ek (azonos):
function cleanOptional(val) { ... }
function cleanCta(cta) { ... }
function isDefined(val) { ... }
function hasRenderableHero(data) { ... }

// Kliens-specifikus:
function normalizeSectionData(type, data) {
  switch (type) {
    case 'eb-hero': return normalizeEbHero(data);
    case 'eb-about': return normalizeEbAbout(data);
    // ...
  }
}
function normalizeEbHero(data) { /* ... */ }
function normalizeEbAbout(data) { /* ... */ }
```

**Javaslat:** Helper-ek → `@spektra/data`. A switch-case + normalizer függvények generálhatók egy section-schema-ból, VAGY: a normalizer marad kliensben, de a helper-eket importálja:

```typescript
import { cleanOptional, cleanCta, hasRenderableHero, normalizeSiteData as baseNormalize } from '@spektra/data';
```

#### B3: builders.php generálás

**Közös minta mindkét kliensben:**
```php
function spektra_build_{prefix}_{section}( string $p, int $pid ): ?array {
    $title = spektra_get_field( $p . 'title', $pid );
    if ( $title === null ) return null;
    $data = [ 'title' => $title ];
    // ... scalar, image, repeater mezők ...
    return $data;
}
spektra_register_section_builder( '{prefix}-{section}', 'spektra_build_{prefix}_{section}' );
```

**Javaslat:** Egy deklaratív PHP registry, ami a field group definíciókból automatikusan generálja a builder-eket:

```php
// sp-infra generic builder (jövő)
spektra_auto_register_builders( $field_groups_dir, $prefix );
// Beolvassa a sections/*.php fájlokat, kinyeri a field definíciókat,
// és generálja a builder függvényeket futásidőben.
```

**Effort:** Közepes-nagy. A jelenlegi builder-ek tartalmaznak finom logikát (CTA feltételes megjelenítés, repeater mapping, kép fallback), ami nehezen generálható.

#### B4: ACF field group PHP generálás

**Ez a legegyszerűbb generálási célpont.** A fájlok 100%-ban templálhatók:

```
Input:  prefix=eb, section=hero, fields=[{name:title, type:text, required:true}, ...]
Output: sections/eb-hero.php (teljes ACF field group definíció)
```

**Jelenlegi struktúra (azonos mindkét kliensben):**
```php
return [
    'key'    => 'group_{prefix}_{section}',
    'title'  => '{PREFIX} {Section}',
    'fields' => [
        [
            'key'   => 'field_{prefix}_{section}_{field}',
            'label' => '{Field Label}',
            'name'  => '{prefix}_{section}_{field}',
            'type'  => '{text|textarea|image|repeater|url|email|...}',
        ],
        // ...
    ],
    'location' => [[['param' => 'page_type', 'operator' => '==', 'value' => 'front_page']]],
    'menu_order' => {N},
];
```

**Effort:** Kis. Egy Node script + Handlebars/EJS template.

---

### Kategória C: Package.json — közös alap

**Tapasztalat:** Az exotica scaffold-nál hiányzott tsx, @types/node, és az ESLint/plugin verziók drifteltek.

**Javaslat:** Monorepo workspace-level shared config:

```jsonc
// sp-platform/packages/client-base/package.json
{
  "name": "@spektra/client-base",
  "peerDependencies": {
    "tsx": "^4.21.0",
    "@types/node": "^25.0.0",
    "typescript": "^5.7.0",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.3.0",
    "vitest": "^2.0.0"
  }
}

// sp-exotica/package.json
{
  "devDependencies": {
    "@spektra/client-base": "workspace:*"
  }
}
```

Vagy: `pnpm` catalog használata a verziók konzisztens kezelésére.

---

### Kategória D: Kliens-specifikus — NEM emelhető ki

| Fájl | Miért marad kliensben |
|------|----------------------|
| `src/data/site.ts` | A teljes tartalmi modell ügyfél-specifikus |
| `src/data/types.ts` | Section típus definíciók az ügyfél modellhez |
| `src/sections/*.tsx` | React komponensek, ügyfél design |
| `src/theme/` | Szín paletta, tipográfia, spacing |
| `.env` | WP URL, port, data source |
| `infra/seed/mapping.ts` | A mező-leképezés (de a helper-ek kivihetők) |

---

## Implementációs terv — prioritás szerint

### P1: Azonnali deduplikáció (1-2 nap, nincs breaking change)

```
sp-platform/packages/
  data/
    src/
      create-adapter.ts      ← export { createAdapter }
      wp-mapper.ts            ← export { mapWordPressResponse, ... }
      normalize-helpers.ts    ← export { cleanOptional, cleanCta, ... }
    package.json              ← @spektra/data
  seed/
    src/
      export-seed.ts          ← export { exportSeed }
      asset-loader.mjs
      register-asset-hooks.mjs
    tsconfig.seed.json        ← base config
    package.json              ← @spektra/seed
```

**Kliens változás:**
```typescript
// Előtte:
import { createWordPressAdapter } from './wordpress-adapter';
// Utána:
import { createWordPressAdapter } from '@spektra/data';
```

**Seed változás:**
```json
// package.json script
// Előtte:
"seed:export": "tsx --import ./infra/seed/register-asset-hooks.mjs infra/seed/export-seed.ts"
// Utána:
"seed:export": "spektra-seed export"  // bin script a @spektra/seed csomagból
```

### P2: ACF field group generátor (2-3 nap)

```
sp-infra/
  scripts/
    generate-acf-fields.ts   ← bemenete: section-schema.yaml
```

**Input:** `section-schema.yaml` (kliensben):
```yaml
prefix: eb
sections:
  hero:
    menu_order: 0
    fields:
      title: { type: text, required: true }
      subtitle: { type: text }
      description: { type: textarea }
      background_image: { type: image }
      background_image_alt: { type: text }
      primary_cta_text: { type: text }
      primary_cta_href: { type: text }
      secondary_cta_text: { type: text }
      secondary_cta_href: { type: text }
  about:
    menu_order: 1
    fields:
      title: { type: text, required: true }
      # ...
```

**Output:** `infra/acf/sections/eb-hero.php`, `eb-about.php`, stb. — teljes PHP fájlok.

### P3: Kliens scaffold CLI (3-5 nap)

```bash
spektra create-client exotica \
  --prefix eb \
  --display-name "Exotica by Bella" \
  --port 5175 \
  --wp-domain exotica.local \
  --sections hero,about,products,animals,gallery,contact
```

**Generál:**
- `package.json` (helyes devDependencies, scripts)
- `vite.config.ts` (port, plugin)
- `.env` (WP URL, data source)
- `infra/acf/sections/*.php` (schema-ból)
- `infra/acf/builders.php` (schema-ból)
- `infra/seed/mapping.ts` (skeleton, extract-ek üresek)
- `src/data/create-adapter.ts` → import from `@spektra/data`
- `src/data/wp-mapper.ts` → import from `@spektra/data`
- `src/data/normalize-site-data.ts` (skeleton switch-case)
- `src/data/types.ts` (skeleton section types)

**Nem generál** (továbbra is kézzel):
- `src/data/site.ts` (tartalom)
- `src/sections/*.tsx` (React komponensek)
- `src/theme/` (vizuális design)
- `infra/seed/mapping.ts` extract függvények (az adat-modell specifikus)

---

## Összefoglaló mátrix

| Terület | Most | P1 után | P2 után | P3 után |
|---------|------|---------|---------|---------|
| create-adapter.ts | Copy-paste | Shared csomag | - | Auto-generated import |
| wp-mapper.ts | Copy-paste | Shared csomag | - | Auto-generated import |
| seed tooling (4 fájl) | Copy-paste | Shared csomag | - | Nincs kliens-fájl |
| ACF PHP-k (sections) | Kézi | Kézi | Generált | Auto-generated |
| builders.php | Kézi | Kézi | Kézi* | Skeleton generált |
| mapping.ts | Kézi | Helper-ek shared | Helper-ek shared | Skeleton generált |
| normalize-site-data.ts | Kézi | Helper-ek shared | Helper-ek shared | Skeleton generált |
| package.json | Kézi + elfelejtett deps | Catalog/base pkg | - | Auto-generated |
| Új kliens idő | ~2 óra | ~1.5 óra | ~1 óra | ~15 perc |

\* A builders.php generálása (P3) korlátozott, mert a builder logika nem triviális — CTA feltételes megjelenítés, repeater mapping, kép fallback mind egyedi logikát tartalmazhat. A skeleton generálás + kézi finomhangolás reálisabb, mint a 100% generálás.

---

## Kockázatok és trade-off-ok

| Kockázat | Valószínűség | Hatás | Mitigáció |
|----------|-------------|-------|-----------|
| Shared csomag breaking change minden kliensre hat | Közepes | Nagy | Szemantikus verziózás, CI tesztek minden kliensben |
| Generátor túl merev, egyedi esetek nem férnek bele | Magas | Közepes | Eject lehetőség: generált fájl felülírható kézzel |
| Monorepo complexity nő | Alacsony | Alacsony | Már most monorepo, a packages/ mappa konvenció |
| YAML schema maintenance | Közepes | Alacsony | A schema a single source of truth — ha módosul, újragenerálás |

---

## Döntési pontok

1. **Mikor kezdjük P1-et?** → A következő kliens előtt ajánlott. Ha a 3. kliens is copy-paste, a debt hármasra nő.
2. **P2 generátor Node vagy PHP?** → Node ajánlott (a seed tooling már Node-ban van, konzisztens).
3. **P3 CLI hol éljen?** → `sp-platform/packages/cli/` vagy `sp-infra/scripts/` — az utóbbi egyszerűbb.
4. **Kell-e a mapping.ts-t YAML-ből generálni?** → Nem feltétlenül. A TypeScript extract függvények túl kifejezőek YAML-hez. A skeleton (üres extract-ekkel) generálható, a kitöltés marad kézi.
