# Phase 8.5 — Content Parity Bootstrap — Log

Kronológikus napló (fordított sorrend — legújabb felül): mi jött létre, mikor, miért.

> Workstream definíció: [content-parity-bootstrap.md](content-parity-bootstrap.md)
> TODO checklist: [content-parity-bootstrap-todo.md](content-parity-bootstrap-todo.md)

---

## Jelenlegi állapot

> Utolsó frissítés: P8.5.6 Documentation — CLOSED (#15)

### Fájlok (sp-docs)

```
sp-docs/
├── content-parity-bootstrap.md       ← workstream definíció (ownership, boundary, guardrail)
├── content-parity-bootstrap-log.md   ← ez a fájl
├── content-parity-bootstrap-todo.md  ← 26 lépéses checklist
└── wp-integration-plan-v4.md         ← §20 roadmap — Phase 8.5 sor hozzáadva
```

### Boundary döntés

| Fájl | Repo | Indoklás |
|---|---|---|
| `infra/seed/export-seed.ts` | sp-benettcar | kliens site.ts-t importálja, kliens mapping-et használja |
| `infra/seed/mapping.ts` | sp-benettcar | ACF field nevek = kliens-specifikus |
| `seed/import-seed.php` | sp-infra | ACF API (update_field) = infra művelet |
| `seed/verify-parity.ts` | sp-infra | parity check = platform-szintű gate |

### Commit history

| # | Hash | Leírás | Mini-phase |
|---|---|---|---|
| 1 | `41b6645` | docs(P8.5.1): architecture closure — workstream def | P8.5.1 |
| 2 | `b64157a` | docs(P8.5.1): add content parity bootstrap TODO checklist | P8.5.1 |
| 3 | `235a6b1` | docs(P8.5.2): v4 seed docs alignment — export ownership + P10.4 alias | P8.5.2 |
| 4 | `38f095d` | fix(P8.5.2): seed README boundary + seed.json gitignore | P8.5.2 |
| 5 | `42e10bb` | docs(P8.5.3): tooling design — seed shape, ACF mapping, CLI interfaces | P8.5.3 |
| 6 | `c4ed7f6` | P8.5.4: ACF alt fields + seed pipeline (mapping.ts, export-seed.ts) | P8.5.4 (sp-benettcar) |
| 7 | `7aa1964` | P8.5.4: ACF alt support + import-seed.php + scaffold cleanup | P8.5.4 (sp-infra) |
| 8 | `9ff505f` | P8.5.4 meta: tooling design updates + artefact fixes | P8.5.4 (sp-docs) |
| 9 | `68357a8` | fix(P8.5.4): make seed export runnable — tsx + asset loader + npm scripts | P8.5.4 (sp-benettcar) |
| 10 | `f9eb596` | fix(P8.5.4): align bootstrap doc with actual seed contract | P8.5.4 (sp-docs) |
| 11 | `4006047` | P8.5.5: verify-parity.ts + dump-acf.php — parity verification tooling | P8.5.5 (sp-infra) |
| 12 | `ef237c0` | fix(P8.5.5): dump-acf.php — reconstruct {url,alt} shape for image fields | P8.5.5 (sp-infra) |
| 13 | `6d94056` | fix(P8.5.5): local asset path fix — relative refs + warnings | P8.5.5 (sp-benettcar) |
| 14 | `d3697a7` | fix(P8.5.5): @types/node + infra/seed/tsconfig.json | P8.5.5 (sp-benettcar) |
| 15 | `60555fa` | P8.5.5: seed-pipeline.ps1 orchestrator + WP-CLI arg compat | P8.5.5 (sp-infra) |

---

## #15 — Pipeline orchestrator + WP-CLI arg fix (2026-04-06) · `60555fa`

**Mini-phase:** P8.5.5 (sp-infra)

### Mi jött létre / változott

**Új fájl:**
- `seed/seed-pipeline.ps1`: Egyparancsos orchestrátor — export → import → dump → verify.
  Auto-selects PHP 8.4.x (8.5+ breaks WP-CLI phar). Auto-downloads wp-cli.phar.
  Flagek: `-DryRun`, `-Verbose`, `-SkipExport`.

**Módosított fájlok:**
- `seed/import-seed.php`: Dual-form arg parsing — `--dry-run` és `dry-run` is elfogadott
  (WP-CLI intercepts `--` prefixed custom args)
- `seed/dump-acf.php`: Dual-form arg parsing — `--output` és `output` is elfogadott
- `.gitignore`: `seed/wp-cli.phar` szabály hozzáadva

### Éles pipeline eredmény

```
Total: 55 fields
  Match:    53
  Mismatch: 2  (bc_service_contact, bc_contact_info — ACF regisztráció hiányzik)
  Missing:  0
```

### Sequencing hibák felfedezve

1. PHP 8.5.0 auto-selection → WP-CLI phar deprecation warning flood → fix: prefer 8.4.x
2. WP-CLI `eval-file` intercepts `--` prefixed args → fix: bare positional arg form
3. PowerShell 5.1 nem parse-ol non-ASCII karaktereket (─, →, ✓) → fix: ASCII-only strings

### Státusz

✅ Pusholva — `60555fa`

---

## #14 — @types/node + infra/seed/tsconfig (2026-04-06) · `d3697a7`

**Mini-phase:** P8.5.5 (sp-benettcar)

### Mi változott

- `package.json`: `@types/node` devDependency hozzáadva
- `infra/seed/tsconfig.json`: Külön tsconfig a seed tooling-hoz (`"types": ["node"]`)
  — nem keveredik a frontend build-del (main tsconfig: `"include": ["src"]`)

### Miért

- Az export-seed.ts `process`, `path`, `fs` Node API-kat használ — @types/node nélkül
  type error-ok az IDE-ben és `tsc --noEmit`-nál.

### Státusz

✅ Pusholva — `d3697a7`

---

## #13 — Local asset path fix (2026-04-06) · `6d94056`

**Mini-phase:** P8.5.5 (sp-benettcar)

### Mi változott

- `infra/seed/asset-loader.mjs`: A fájlrendszer-path (`/D:/Projects/...`) helyett
  projekt-relatív path-t ad vissza (`src/assets/brands/vw-logo.jpg`).
  A PROJECT_ROOT-ot a loader saját pozíciójából számolja (`../../`).
- `infra/seed/export-seed.ts`: `detectLocalAssets()` függvény — a seed mezőkön
  végig megy és figyelmeztet minden nem-URL képreferenciára.
  6 bc-brand logó flagelve, 0 törött abszolút path.

### Miért

- A site.ts ES import-okat használ a brand logókhoz (`import vwLogo from '...jpg'`).
- A Vite dev/build ezeket asset URL-ekké oldja fel.
- A seed export kontextusban (tsx + asset-loader) korábban Windows abszolút
  fájlrendszer-pathok kerültek a seed.json-ba — a WP-ben nem működnek.
- A parity check átmehetett volna (mindkét oldal törött path), de a WP frontend
  törött `<img src>` URL-eket kapott volna.
- A relatív path + web URL-re cserélés P11+ scope (media library upload).

### Státusz

✅ Pusholva — `6d94056`

**Mini-phase:** P8.5.5 (sp-infra)

### Mi változott

- `seed/dump-acf.php`: Image mezők esetén (ahol a seed érték `{url, alt}` object)
  a dump során a WP-ből olvasott URL string + a companion `_alt` mező értéke
  vissza lesz rakva `{url, alt}` shape-be a wp-state.json-ban.

### Miért

- Az import-seed.php az image `{url, alt}` object-ből kiszedi az URL-t és stringként
  tárolja `update_field()`-del. Az alt külön `_alt` mezőben megy.
- A dump-acf.php korábbi verziója nyersen olvasta vissza → plain URL string.
- A verify-parity.ts az image mezőknél `{url, alt}` shape-et vár mindkét oldalról.
- Round-trip shape mismatch → garantált FAIL a hero/about image mezőkön.

### Státusz

✅ Pusholva — `ef237c0`

**Mini-phase:** P8.5.5 (sp-infra)

### Mi jött létre / változott

**Új fájlok:**
- `seed/verify-parity.ts`: Parity check tool — összehasonlítja seed.json-t és wp-state.json-t.
  Text/CTA: exact match. Image: structural match (URL eltérhet, alt egyeznie kell).
  `--verbose`, `--strict` flagek. Exit 0=PASS, 1=FAIL.
- `seed/dump-acf.php`: WP-CLI eval-file script — kiolvassa a seed.json kulcsainak megfelelő
  ACF mezőket és WP optionst, wp-state.json-be írja.

**Módosított fájlok:**
- `seed/package.json`: verify/verify:verbose scriptek, régi export script törölve
- `seed/README.md`: verification workflow szekció, frissített struktúra
- `.gitignore`: `seed/wp-state.json` szabály hozzáadva

### Miért

- A P8.5.5 célja a teljes verify pipeline: seed.json → WP import → dump → parity check.
- A verify-parity.ts támogat seed és wp-state.json összevetést a §4.1 parity szabályok szerint.
- A dump-acf.php biztosítja, hogy a WP állapotot azonos struktúrában kapjuk vissza mint a seed.
- seed.json generálva — 53 mező, 10 szekció — dry-run és real export is PASS.

### WP futás — manuális gate

A tényleges WP import + dump + parity check élő WordPress-t igényel:
```bash
wp eval-file import-seed.php          # seed → WP DB
wp eval-file dump-acf.php             # WP DB → wp-state.json
npx tsx verify-parity.ts --verbose     # PASS / FAIL
```

### Státusz

✅ Pusholva — `4006047`
⏳ WP import + parity verify PASS — manuális gate, élő WP-t igényel

**Mini-phase:** P8.5.4 (sp-docs)

### Mi változott

- `content-parity-bootstrap.md` §2: pipeline import-seed.sh → import-seed.php (wp eval-file)
- `content-parity-bootstrap.md` §3.2: boundary tábla import-seed.sh → import-seed.php
- `content-parity-bootstrap.md` §6.1: seed shape átírás — sections-grouped/kind-annotated → flat `fields` object
- `content-parity-bootstrap.md` §6.1 szabályok: kind annotáció → `detect_kind()` value shape alapú

### Miért

- [P2 finding] A doksi §6.1 példa JSON-ja `sections → bc-hero → { value, kind }` struktúrát írt,
  miközben az export-seed.ts valójában lapos `fields` objektumot gyárt, az import-seed.php
  pedig `detect_kind()`-dal határozza meg a típust az érték shape-jéből.
- Az import-seed.sh → .php váltás még nem volt végigírt a pipeline (§2) és boundary (§3.2) szekciókban.

### Státusz

✅ Pusholva — `f9eb596`

---

## #9 — Seed export futtathatóvá téve (2026-04-06) · `68357a8`

**Mini-phase:** P8.5.4 (sp-benettcar)

### Mi jött létre / változott

- `package.json`: tsx devDependency + `seed:export` / `seed:export:dry` npm scriptek
- `infra/seed/asset-loader.mjs`: Node ESM loader hook — image importokat file path stringgé stub-olja
- `infra/seed/register-asset-hooks.mjs`: `module.register()` bootstrap a loader hook-hoz

### Miért

- [P1 finding] Az export-seed.ts `npx tsx`-re épült, de tsx nem volt devDep — npx külso
  registry-ről próbálta húzni, failing.
- A site.ts image importokat használ (.jpg/.png), amiket Vite kezel build-ben, de bare
  tsx/Node nem tud. Az asset-loader.mjs ezeket file path stringgé oldja fel.
- Dry-run eredmény: 53 mező, 10 szekció, helyes shape — P8.5.5 unblokkolt.

### Státusz

✅ Pusholva — `68357a8`

---

## #8 — P8.5.4 meta: tooling design updates + artefact fixes (2026-04-07) · `9ff505f`

**Mini-phase:** P8.5.4 (sp-docs)

### Mi változott

- `content-parity-bootstrap.md` §6.2: brands[N]/content[N] shape korrekció (site.ts string[] → ACF transform)
- `content-parity-bootstrap.md` §6.4: import-seed.sh → import-seed.php (ACF API, wp eval-file)
- `content-parity-bootstrap.md` §6.5: alt field bővítés dokumentálva (hero/about/team)
- `content-parity-bootstrap.md` §6.6: navigation kizárva parity scope-ból
- `content-parity-bootstrap-todo.md`: import-seed.sh → import-seed.php referenciák javítva
- `content-parity-bootstrap-log.md`: import-seed.sh → import-seed.php #5 entry-ben

### Miért

- A §6.2 tábla brands[N].name / content[N].paragraph alakot használt, de a site.ts-ben string[] van,
  nem { name: string }[]. A mapping.ts helyes transzformációt végez, a táblát is korrigálni kellett.
- Az import-seed.sh → import-seed.php váltás következményeként minden meta-dokumentumot is frissíteni kellett.

### Státusz

✅ Pusholva — `9ff505f`

---

## #7 — ACF alt support + import-seed.php + scaffold cleanup (2026-04-07) · `7aa1964`

**Mini-phase:** P8.5.4 (sp-infra)

### Mi változott

- `acf/media-helper.php`: `spektra_normalize_media()` kap `string $alt_override = ''` második paramétert.
  URL string path-nál és ACF image array path-nál is alkalmazza az override-ot.
- `acf/sections.php`: `spektra_build_bc_hero()`, `spektra_build_bc_about()`, `spektra_build_bc_team()`
  átadják a megfelelő `_alt` mezőt a `spektra_normalize_media()`-nak.
- `seed/import-seed.php`: Generikus ACF-aware importer — `update_field()` API-t használ.
  detect_kind() auto-detektálja: scalar, repeater, group, image. `wp eval-file` futtatású.
- `seed/export-seed.ts`: git rm — deprecated scaffold törlése (capability sp-benettcar-ba költözött)
- `seed/README.md`: pipeline + structure frissítve az új boundary-nek + import-seed.php-nek megfelelően

### Miért

- **alt support**: Image-only URL seed → üres alt forever. A bc-hero, bc-about, bc-team ACF group-oknak
  nincs külön alt mező → hozzá kell adni sp-benettcar-ban + sp-infra builder-ekben pass-through.
- **import-seed.php**: `wp post meta update` nem bizonyított repeater/group/image ACF mezőkre.
  Az `update_field()` API natívan kezeli a row count-ot, sub-field indexelést, és a serialize formátumot.
- **scaffold törlés**: A P8.5.1-ben DEPRECATED-ként jelölt sp-infra/seed/export-seed.ts most törölve.

### Státusz

✅ Pusholva — `7aa1964`

---

## #6 — ACF alt fields + seed pipeline (2026-04-07) · `c4ed7f6`

**Mini-phase:** P8.5.4 (sp-benettcar)

### Mi jött létre / változott

**Új fájlok:**
- `infra/seed/mapping.ts`: 10 szekció ACF field mapping — kind-annotált (`scalar`, `repeater`, `group`, `image`).
  Minden szekció extract függvénye a site.ts tényleges shape-jéből dolgozik.
- `infra/seed/export-seed.ts`: CLI tool — importálja site.ts + mapping.ts, generálja seed.json-t.
  Flags: `--output/-o`, `--dry-run`, `--verbose`. Default output: `../../../../sp-infra/seed/seed.json`.

**Módosított fájlok:**
- `infra/acf/sections/bc-hero.php`: `background_image_alt` text mező hozzáadva
- `infra/acf/sections/bc-about.php`: `image_alt` text mező hozzáadva
- `infra/acf/sections/bc-team.php`: members repeater `image_alt` sub-field hozzáadva

### Miért

- A mapping.ts a §6.2 tábla implementációja — a ~80 ACF mező kind-annotáltan,
  helyes extract függvényekkel, a site.ts valós shape-jével dolgozik.
- Az export-seed.ts a §6.3 interface implementációja.
- Az ACF alt mezők a P2 finding megoldása: URL-only seed-ből üres alt jönne,
  ha nincs külön mező ami az alt szöveget tárolja.

### Döntések

1. **Navigation kizárva**: Response Builder config.php-ből olvassa, nem WP DB-ből. Parity scope-on kívül.
2. **import-seed.php, nem .sh**: ACF `update_field()` API, nem `wp post meta update`. PHP, nem bash.
3. **mapping.ts kind annotation**: Minden mező explicit `FieldKind` — az import-seed.php `detect_kind()`-ja is
   ezt a sémát követi, bár a PHP oldalon runtime shape detection van.

### Státusz

✅ Pusholva — `c4ed7f6`

---

## #5 — Tooling Design (2026-04-06) · `42e10bb`

**Mini-phase:** P8.5.3

### Mi jött létre

`content-parity-bootstrap.md` §6 — 5 új szekció:
- §6.1 seed.json shape spec (példa JSON-nal)
- §6.2 ACF field mapping tábla (mind 10 bc-* szekció, ~80 mező)
- §6.3 export-seed CLI interface
- §6.4 import-seed.php interface (ACF API, wp eval-file)
- §6.5 Image handling / media exception spec

### Miért

- A P8.5.4 implementációhoz pontos specifikacíó kell: milyen formátumban írjuk a seed.json-t,
  milyen ACF field neveket használunk, hogyan kezeljük a képeket.
- A mapping tábla a `sp-infra/acf/sections.php` builder függvényeiből lett összeállítva —
  minden ACF field név megegyezik a PHP `spektra_get_field()` hívásokban használttal.
- Media exception: image URL-ek stringként mennek a seed-be, nem WP attachment ID-ként.
  A `spektra_normalize_media()` URL stringet is elfogad.

### Döntések

1. **Nincs `--input` / `--mapping` flag** — az export a kliens repóban van, relative import elég
2. **`post_id: "front_page"`** — import-seed.php runtime-ban feloldja (`get_option('page_on_front')`)
3. **Repeater/Group** — `update_field()` kezeli a row count-ot és sub-field indexelést
4. **Boolean**: `"1"` / `"0"` (WP meta convention)
5. **Image**: URL string (nem attachment ID) — media library sync P11+ scope

### Státusz

✅ Pusholva — `42e10bb`

---

## #4 — sp-infra seed alignment (2026-04-06) · `38f095d`

**Commit:** `fix(P8.5.2): seed README boundary alignment + seed.json gitignore`

**Repo:** sp-infra

### Mi változott

- `seed/README.md`: Pipeline diagram + structure frissítve az új boundary-nek megfelelően.
  Export-seed.ts jelölve mint DEPRECATED scaffold (törlendő P8.5.4-ben).
  Új strucúra: import-seed.sh + verify-parity.ts hozzáadva.
- `.gitignore`: `seed/seed.json` szabály hozzáadva — korábban dokumentált, de hiányzó.

### Miért

- [P2 finding] A seed README továbbra is sp-infra/seed/export-seed.ts-t írta kanonikus exportként,
  miközben a P8.5 boundary döntés ezt a kliens repóba helyezte.
- [P2 finding] A seed.json gitignore-állítás dokumentálva volt (README + content-parity-bootstrap.md),
  de a .gitignore fájlból ténylegesen hiányzott. Operatív rés, nem csak wording drift.

### Státusz

✅ Pusholva — `38f095d`

---

## #3 — v4 plan seed docs alignment (2026-04-06) · `235a6b1`

**Commit:** `docs(P8.5.2): align v4 seed docs with P8.5 boundary — export ownership + P10.4 alias`

**Repo:** sp-docs

### Mi változott

- `wp-integration-plan-v4.md` §13.2: `sp-infra/seed/export-seed.ts` → `<client>/infra/seed/export-seed.ts`.
  P8.5 boundary döntés note hozzáadva, link a content-parity-bootstrap.md §3.2-re.
- `wp-integration-plan-v4.md` §20 P10.4: áthúzva, legacy alias note hozzáadva
  ("→ Phase 8.5-be áthelyezve (P8.5.4–P8.5.5)").

### Miért

- [P2 finding] A v4 plan §13.2 még sp-infra-t írt kanonikus export helyként — két párhuzamos
  source of truth ugyanarra a capability-re.
- [P3 finding] P10.4 és P8.5.4–P8.5.5 dupla tracking volt explicit kapcsolat nélkül.
  Most P10.4 egyértelműen aliasként jelölve.

### Státusz

✅ Pusholva — `235a6b1`

---

## #2 — TODO checklist (2026-04-06) · `b64157a`

**Commit:** `docs(P8.5.1): add content parity bootstrap TODO checklist`

### Mi jött létre

```
sp-docs/
└── content-parity-bootstrap-todo.md   ← 26 lépéses checklist (P8.5.1–P8.5.6)
```

### Miért

- A Phase 8.5 workstream 6 mini-fázisból áll, összesen 26 egyedi lépéssel
- Szükség van egy trackelhető checklist-re, ami a mini-fázisok előrehaladását mutatja
- P8.5.1 (Architecture Closure) 4 lépése már kipipálva

### Státusz

✅ Pusholva — `b64157a`

---

## #1 — Architecture Closure (2026-04-06) · `41b6645`

**Commit:** `docs(P8.5.1): architecture closure — content parity bootstrap workstream`

### Mi jött létre

```
sp-docs/
├── content-parity-bootstrap.md       ← workstream definíció
├── content-parity-bootstrap-log.md   ← log scaffold
└── wp-integration-plan-v4.md         ← Phase 8.5 sor a §20 roadmap-ben
```

### Miért

- Phase 8.3-ban bevezettük az adapter switcht (JSON → WP), de a seed pipeline (P10.4) még 2 fázissal arrébb van
- A WP adatbázisban dummy tartalom van (P6.4-ből), miközben a `site.ts` valós, éles tartalom
- Phase 8.5 interleaved workstream szükséges a content parity biztosításához
- A v4 roadmap fázisszámozás NEM változik — 8.5 egy beszúrt mini-fázis

### Döntések

1. **Canonical content source**: `site.ts` (sp-benettcar) — egyetlen igazság forrása
2. **Export boundary**: export-seed.ts + mapping.ts → sp-benettcar/infra/seed/ (kliens-specifikus)
3. **Import boundary**: import-seed.sh → sp-infra/seed/ (infra művelet)
4. **Verify boundary**: verify-parity.ts → sp-infra/seed/ (platform-szintű gate)
5. **Guardrail**: CMS Runtime Parity Gate — adapter switch tilos parity nélkül
6. **sp-infra/seed/export-seed.ts** scaffold: P8.5.4-ben törlendő (rossz helyen van)
7. **Phase 9 blokkolás**: Phase 9 NEM blokkolt, de production cutover tilos P8.5.5 PASS nélkül

### Státusz

✅ Pusholva — `41b6645`
