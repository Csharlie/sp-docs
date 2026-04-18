# Phase 12.3 — First Runtime Migration Planning / Pilot

## Purpose

Az első runtime migration target kiválasztása az ACF Repeater dependency kiváltására a free WordPress baseline-ban. Ez planning fázis — runtime implementáció csak az elfogadott target döntés után indulhat.

## Context

P12.2 bevezette a Repeatable Content Source Strategy-t (`concepts/repeatable-content-source-strategy.md`):
- `cpt_collection` — default free baseline (CPT; taxonomy csak ahol kategorizáció indokolt)
- `fixed_slots` — bounded kis tartalom (ACF Free-kompatibilis)
- `acf_repeater_optional` — opcionális jövőbeli Pro source

13 repeater field / 11 szekció classifikálva. Stratégiai döntés: Option C (Split Tracks) — lásd Decision szekció.

---

## Decision

Option C selected: Split Tracks.

- **P12.3a**: fixed_slots technical pilot planning
- **P12.3b**: BenettCar cpt_collection design

**Rationale:**
- fixed_slots pilot alacsony kockázatú validációt ad az ACF Repeater kiváltásra
- BenettCar cpt_collection design védi a handover prioritást
- split track nem blokkolja a platform validációt, miközben a valós kliens editorial pain-t is címzi

**Fontos:** Ez a lépés a stratégiai döntést rögzíti. Egyik track sem tartalmaz runtime implementációt. P12.3a a planning lezárásakor implementációs promptot produkál. P12.3b design dokumentum marad az explicit platform-owner jóváhagyásig.

---

## Decision Context (resolved)

Az alábbi szekciók a döntési folyamat dokumentációja. Option C kiválasztásra került — a candidate-ok és kritériumok a döntés indoklását szolgálják.

### Candidate A — fixed_slots Technical Pilot (→ P12.3a)

**Lehetséges target-ek:**
- sp-exotica: `eb-contact.opening_hours`
- sp-exotica: `eb-about.values`

**Indoklás:**
- Kis, bounded tartalom — max 5-7 item
- Nincs image handling
- Nincs taxonomy szükséglet
- Lowest-risk validáció az ACF Repeater kiváltásra
- Alkalmas a `fixed_slots` pattern bizonyítására

**Kockázat:**
- Alacsonyabb üzleti impact mint a BenettCar handover kérdések
- Nem validálja a `cpt_collection` pattern-t
- Exotica-specifikus — nem ad közvetlenül BenettCar handover értéket

### Candidate B — BenettCar cpt_collection Planning (→ P12.3b)

**Lehetséges target-ek:**
- sp-benettcar: `bc-gallery.images`
- sp-benettcar: `bc-brand.brands`
- sp-benettcar: `bc-team.members`

**Fontos: Candidate B design és migration planning dokumentum. NEM jelent CPT runtime kód implementációt — a végleges target kiválasztás explicit platform-owner jóváhagyást igényel.**

**Indoklás:**
- Közvetlen BenettCar handover stabilitás támogatás
- Valós editorial UX pain megoldása
- A fontosabb `cpt_collection` pattern validálása

**Kockázat:**
- Nagyobb scope
- CPT naming convention döntés szükséges
- Collection loading pattern definiálása szükséges
- Seed strategy planning szükséges
- Erősebb parity validation szükséges

### Option C — Split Tracks (selected)

Option C kiválasztva. A split tracks megközelítés:
- **P12.3a**: `fixed_slots` pilot planning — ez záródik előbb.
- **P12.3b**: BenettCar `cpt_collection` design párhuzamosan indulhat, de design dokumentum marad az explicit jóváhagyásig.
- P12.3b NEM blokkolhatja P12.3a lezárását.
- Runtime implementáció továbbra is külön elfogadott implementációs promptot igényel.

### Decision Criteria (applied)

Kandidátusok összehasonlítása az alábbi dimenziók mentén történt:
- technikai kockázat
- üzleti / handover impact
- implementációs scope
- SiteData stabilitási kockázat
- seed pipeline impact
- editorial UX javulás
- jövőbeli scaffold/generator érték

**Döntési felelősség:**
A végső target döntés nem automatikus, ha tradeoff-ok fennállnak. A tradeoff-ot világosan kell bemutatni és egy opciót ajánlani. A végleges kiválasztás platform-owner döntés, kivéve ha az evidencia egyértelmű.

**Súlyozási irányelvek:**
- Delivery/handover fókusz: editorial UX improvement és business impact súlyozottabb
- Platform validáció fókusz: technikai kockázat és bounded scope súlyozottabb
- Hosszú távú platform érettség: scaffold/generator érték és seed impact súlyozottabb

**Eredmény:** Option C kiválasztva — mindkét dimenzió (platform validáció + handover) párhuzamosan haladhat.

---

## P12.3a — fixed_slots Technical Pilot Planning

**Selected target:** `eb-about.values` (sp-exotica)

**Indoklás:**
- Valódi, látható adat renderel a dev frontenden (3 value string)
- Érdemi parity validation — van mit összehasonlítani before/after
- 1 text sub-field repeater, `string[]` SiteData output — minimális komplexitás
- Normalizer egyszerű (nincs hard gate, csak whitespace cleanup)
- `eb-contact.opening_hours` strukturálisan identikus, de az alap frontenden nem renderel tartalmat — nem ad érdemi validációt

**Evaluated candidates:**
- sp-exotica: `eb-about.values` — SELECTED
- sp-exotica: `eb-contact.opening_hours` — elvetett (nincs látható tartalom az alap frontenden)

**Purpose:**
- fixed_slots source strategy bizonyítása
- egy egyszerű ACF Repeater dependency kiváltásának előkészítése ACF Free-kompatibilis bounded mezőkkel
- SiteData output változatlan marad

**Fontos:** Ez kizárólag planning. P12.3a akkor záródik, amikor az implementációs prompt elkészült. A tényleges runtime implementáció külön explicit implementációs promptot igényel.

**Current runtime shape (before):**
- PHP: `eb_about_values` repeater, sub-field `text` (text, required)
- Schema: `values?: string[]`
- Mapper: `asArray(raw.values).map(asString).filter(isDefined)`
- Seed: `kind: 'repeater'`, `string[] → { text: string }[]`
- Component: `values.map(v => <li>✦ {v}</li>)`
- Dev data: 3 items

**Field representation döntés:** Textarea + Split

- **ACF field:** egyetlen `eb_about_values_text` textarea mező váltja a repeater-t
- **Builder mapping:** `textarea.split(/\r?\n/).map(s => s.trim()).filter(Boolean)` → `string[]`
- **SiteData output:** változatlan `values: string[]`
- **Seed:** egyetlen `update_field('eb_about_values_text', values.join('\n'))` hívás
- **Editorial UX:** 1 textarea többsoros szerkesztéssel, soronként 1 érték

**Elvetett alternatívák:**
- Numbered fields (7× `eb_about_value_N`) — zajos admin UX, hardcoded maximum
- JSON textarea — rossz editorial UX, hibalehetőség

### Builder Mapping Plan

**Felfedezett bug:** A jelenlegi `mapEbAbout` TS mapper `asString()` hívást végez a repeater row objektumokon (`{text: "..."}`), ami `undefined`-ot ad, mivel `typeof {text: "..."} !== 'string'`. A dev frontend a `site.ts`-ből kapja a `string[]`-t közvetlenül, de a WP REST endpoint-ról az adatok elvesznének. A textarea+split migration ezt a pre-existing bug-ot is fixálja.

**Érintett fájlok és változások:**

| # | Fájl | Repó | Változás |
|---|------|------|----------|
| 1 | `infra/acf/sections/eb-about.php` | sp-exotica | repeater → textarea field def |
| 2 | `infra/acf/builders.php` | sp-exotica | split logic a builder-ben |
| 3 | `src/data/wp-mapper.ts` | sp-exotica | NEM változik (already works with `string[]`) |
| 4 | `infra/seed/mapping.ts` | sp-exotica | repeater extract → scalar join |
| 5 | `src/data/normalize-site-data.ts` | sp-exotica | NEM változik |
| 6 | `src/data/site.ts` | sp-exotica | NEM változik |
| 7 | `src/sections/eb-about/eb-about.schema.ts` | sp-exotica | NEM változik |
| 8 | `src/sections/eb-about/eb-about.component.tsx` | sp-exotica | NEM változik |

**Változás #1 — ACF field definition** (`eb-about.php`):

Before:
```php
'key'  => 'field_eb_about_values',
'name' => 'eb_about_values',
'type' => 'repeater',
'sub_fields' => [[ 'key' => 'field_eb_about_values_text', 'name' => 'text', 'type' => 'text' ]]
```

After:
```php
'key'   => 'field_eb_about_values_text',
'label' => 'Values (soronként 1 érték)',
'name'  => 'eb_about_values_text',
'type'  => 'textarea',
'rows'  => 5,
'new_lines' => '',  // no <br> or <p> wrapping
```

**Változás #2 — PHP builder** (`builders.php`):

Before:
```php
'values' => spektra_get_field( $p . 'values', $pid, [] ),
```

After:
```php
'values' => spektra_split_textarea( spektra_get_field( $p . 'values_text', $pid, '' ) ),
```

Ahol `spektra_split_textarea` (sp-infra helper vagy lokális):
```php
function spektra_split_textarea( string $text ): array {
    if ( $text === '' ) return [];
    return array_values( array_filter( array_map( 'trim', explode( "\n", $text ) ) ) );
}
```

**Változás #3 — TS mapper**: Nem szükséges. A mapper `asArray(raw.values).map(asString).filter(isDefined)` — ha a PHP builder `string[]`-t ad, a mapper helyesen dolgozza fel. A migration side-effect: fixálja a pre-existing bug-ot.

**Változás #4 — Seed mapping** (`mapping.ts`):

Before:
```typescript
{ acfKey: 'eb_about_values', kind: 'repeater',
  extract: (d) => { const values = d.values as string[]; return values.map(text => ({text: str(text)})) } }
```

After:
```typescript
{ acfKey: 'eb_about_values_text', kind: 'scalar',
  extract: (d) => { const values = d.values as string[]; return values.join('\n') } }
```

**Data flow after migration:**

| Lépés | Formátum | Hol |
|-------|----------|-----|
| WP admin | textarea: `"line1\nline2\nline3"` | ACF textarea |
| PHP builder | `spektra_split_textarea()` → `["line1","line2","line3"]` | builders.php |
| REST endpoint | `{"values": ["line1","line2","line3"]}` | JSON output |
| TS mapper | `asArray().map(asString).filter()` → `string[]` | wp-mapper.ts |
| SiteData | `values: string[]` | **változatlan** |
| Component | `values.map(v => <li>✦ {v}</li>)` | **változatlan** |

---

### Seed Impact

**Jelenlegi seed flow (repeater):**
1. `site.ts` → `values: ["V1", "V2", "V3"]` (string array)
2. `export-seed.ts` → `mapping.extract()` → `[{text: "V1"}, {text: "V2"}, {text: "V3"}]`
3. `seed.json` → `"eb_about_values": [{text: "V1"}, ...]`
4. `import-seed.php` → `detect_kind()` → `'repeater'` → `update_field('eb_about_values', [...])`

**Migrált seed flow (textarea):**
1. `site.ts` → `values: ["V1", "V2", "V3"]` — **változatlan**
2. `export-seed.ts` → `mapping.extract()` → `"V1\nV2\nV3"` (joined string)
3. `seed.json` → `"eb_about_values_text": "V1\nV2\nV3"`
4. `import-seed.php` → `detect_kind()` → `'scalar'` → `update_field('eb_about_values_text', "V1\nV2\nV3")`

**Impact elemzés:**
- `site.ts`: NEM változik (továbbra is `string[]`)
- `mapping.ts`: extract function egyszerűsödik (join vs map-to-object)
- `seed.json`: kisebb output (egy string vs array-of-objects)
- `import-seed.php`: `detect_kind()` `'scalar'`-t ad → egyszerűbb path, nincs row iteration
- **sp-infra seed tooling NEM igényel módosítást** — a `detect_kind()` és az `update_field()` path scalar-okat is kezel

**Kockázat:** Nincs. A seed pipeline scalar field-eket natívan támogatja. A `detect_kind()` automatikusan 'scalar'-t ad egy plain string-re.

---

**Required planning output implementáció előtt:**
- ~~kiválasztott fixed_slots target~~ DONE: `eb-about.values`
- ~~field representation döntés~~ DONE: textarea + split
- ~~builder mapping plan~~ DONE
- ~~seed impact~~ DONE
- SiteData parity validation plan
- rollback plan
- implementációs prompt készültségi értékelés

---

### SiteData Parity Validation Plan

**Cél:** Bizonyítani, hogy a migration után a SiteData output byte-ra megegyezik a migration előttivel.

**Validation lépések:**

1. **Before snapshot**: A migration előtt rögzíteni a `/wp-json/spektra/v1/site` endpoint `eb-about` szekciójának JSON outputját
   ```bash
   wp eval-file import-seed.php        # seed betöltés repeater formátumban
   curl -s .../wp-json/spektra/v1/site | jq '.sections[] | select(.type == "eb-about")' > before.json
   ```

2. **Migration végrehajtás**: ACF field definition csere, builder update, seed mapping update

3. **Re-seed**: Az új seed formátummal (textarea) újraseedélni
   ```bash
   wp eval-file import-seed.php        # seed betöltés textarea formátumban
   ```

4. **After snapshot**: Ugyanaz az endpoint
   ```bash
   curl -s .../wp-json/spektra/v1/site | jq '.sections[] | select(.type == "eb-about")' > after.json
   ```

5. **Diff**:
   ```bash
   diff before.json after.json
   ```
   **Elvárt eredmény:** 0 diff — a `values` array tartalma és sorrendje azonos.

**Amit összehasonlítunk:**
- `values` array length (3)
- `values` array tartalom (exact string match minden elemre)
- `values` array sorrend
- Egyéb `eb-about` mezők változatlanok (title, subtitle, description, story, image)

**Amit NEM hasonlítunk össze:**
- ACF admin felületi megjelenés (ez UX change, nem data change)
- wp_postmeta belső szerkezet (ACF implementation detail)
- seed.json formátum (ez szándékosan változik)

**Pass/Fail kritérium:** `before.json` és `after.json` 100% egyezés az `eb-about` szekción belül. Bármi eltérés FAIL → rollback.

---

### Rollback Plan

**Trigger:** A parity validation FAIL, VAGY bármi váratlan viselkedés a migration után.

**Rollback lépések (sorrendben):**

1. **ACF field definition**: `eb-about.php`-ban a textarea field visszacserélése repeater-re (git revert)
2. **Builder**: `builders.php`-ban `spektra_split_textarea()` hívás visszacserélése `spektra_get_field('values', ...)` hívásra (git revert)
3. **Seed mapping**: `mapping.ts`-ben a scalar extract visszacserélése repeater extract-re (git revert)
4. **Re-seed**: Repeater formátumú seed újrafuttatása
   ```bash
   wp eval-file import-seed.php
   ```
5. **Verification**: Before snapshot-tal való egyezés újra ellenőrzése

**Rollback scope:** Kizárólag sp-exotica + sp-infra (ha helper hozzáadva). Más kliensek NEM érintettek.

**Rollback mechanizmus:** `git revert <migration-commit>` — egyetlen commit revertálása elegendő, ha minden migration változás egy commitban van.

**Fontos:** A migration commitot úgy kell megírni, hogy egyetlen `git revert` működjön. Nem szabad más, nem-migration változásokat összekeverni a committal.

---

### Implementációs Prompt Készültségi Értékelés

**Checklist:**

| # | Elem | Státusz | Megjegyzés |
|---|------|---------|------------|
| 1 | Target kiválasztás | ✅ | `eb-about.values` |
| 2 | Field representation | ✅ | textarea + split |
| 3 | Érintett fájlok listája | ✅ | 4 fájl módosul, 4 változatlan |
| 4 | Before/after kód minden érintett fájlhoz | ✅ | PHP ACF, PHP builder, TS seed mapping |
| 5 | SiteData parity validation plan | ✅ | endpoint snapshot diff |
| 6 | Rollback plan | ✅ | single commit revert |
| 7 | Pre-existing bug dokumentálva | ✅ | mapper `asString` on repeater rows |
| 8 | Seed flow before/after | ✅ | repeater → scalar |
| 9 | Nem-érintett fájlok explicitek | ✅ | schema, component, normalizer, site.ts |

**Eredmény: READY**

Az implementációs prompt biztonságosan megírható. Minden szükséges kontextus rendelkezésre áll:
- Pontosan 4 fájl módosítandó (ACF def, builder, seed mapping + opcionálisan sp-infra helper)
- A SiteData output nem változik
- A parity validation terv létezik
- A rollback terv létezik
- A pre-existing mapper bug a migration side-effect-jeként fixálódik

**Döntés szükséges az implementáció előtt:**
- ~~A `spektra_split_textarea()` helper sp-infra-ba kerüljön (reusable) vagy sp-exotica lokális legyen?~~ **RESOLVED: lokális sp-exotica.** Indoklás: pilot migration, egyetlen consumer, kisebb blast radius. Extraction rule: sp-infra-ba emelés csak akkor, ha legalább egy második migration ugyanazzal a split-semantikával ténylegesen megjelenik.

---

**Required planning output összesítés:**
- ~~kiválasztott fixed_slots target~~ DONE: `eb-about.values`
- ~~field representation döntés~~ DONE: textarea + split
- ~~builder mapping plan~~ DONE
- ~~seed impact~~ DONE
- ~~SiteData parity validation plan~~ DONE
- ~~rollback plan~~ DONE
- ~~implementációs prompt készültségi értékelés~~ DONE: **READY**

---

## P12.3b — BenettCar cpt_collection Design [DESIGN COMPLETE]

**Selected target:** `bc-services.services` (sp-benettcar)

**Scope kiegészítés:** Az eredeti Candidate B lista (`bc-gallery`, `bc-brand`, `bc-team`) nem tartalmazta a `bc-services`-t, de a P12.2 elfogadott classifikáció alapján `bc_services_services` is `cpt_collection` stratégiájú. Az alacsonyabb implementációs kockázat (nincs image, nincs taxonomy, 3 scalar mező) miatt ez a legalkalmasabb első cpt_collection pilot target.

**TODO:**
- [x] Első target ajánlás: `bc-services.services`
- [x] CPT naming convention javaslat: `sp_bc_service` (14 char, WP 20 char limit alatt)
- [x] Collection loading pattern javaslat: kliens-lokális `spektra_bc_get_services()`
- [x] Builder output shape: változatlan SiteData `[{ title, icon, description }]`
- [x] Seed strategy impact: `kind: cpt_collection` → P12.5 scope
- [x] SiteData parity validation plan: REST endpoint byte-exact match
- [x] Migration/rollback plan: commit revert + CPT post delete + repeater fallback
- [x] Handover impact assessment: editorial workflow változás (inline repeater → CPT admin menüpont)
- [ ] Implementációs prompt készültségi értékelés: runtime implementáció külön explicit promptot igényel

**Design dokumentum:** `implementation/p12-3b-benettcar-cpt-collection-design.md`

**Deferred targets (komplexitás szerint):**
1. `bc-brand.brands` — image/logo handling
2. `bc-team.members` — image + contact fields
3. `bc-gallery.images` — image + category/taxonomy (legkomplexebb)

**Fontos:** Ez kizárólag design. CPT runtime kód NEM implementálható P12.3b-ben. Runtime implementáció külön explicit promptot igényel. A végleges cpt_collection target kiválasztás explicit platform-owner jóváhagyást igényel implementáció előtt.

---

## Guardrails

- WordPress/CPT/ACF logika NEM kerülhet az sp-platform core-ba
- SiteData shape változatlan marad
- Frontend section props változatlan marad
- ACF Pro nem baseline requirement
- Meglévő kliensek megtarthatják a repeater-eket a migration-ig
- P12.3a planning-only, amíg külön implementációs prompt nem elfogadott
- P12.3b design-only, amíg később expliciten nem jóváhagyott
- Runtime kód csak elfogadott implementációs prompt után indulhat

---

## Done State

P12.3 planning lezárási feltétel:
1. Option C rögzítve mint kiválasztott stratégia
2. P12.3a target ajánlás dokumentálva és implementációs prompt biztonságosan megírható
3. P12.3b cpt_collection design ajánlás dokumentálva
4. SiteData stability validation plan mindkét track-re létezik
5. Rollback plan mindkét track-re létezik
6. Runtime kód NEM módosult ebben a planning lépésben
