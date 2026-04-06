# Phase 8.5 — Content Parity Bootstrap — Log

Kronológikus napló (fordított sorrend — legújabb felül): mi jött létre, mikor, miért.

> Workstream definíció: [content-parity-bootstrap.md](content-parity-bootstrap.md)
> TODO checklist: [content-parity-bootstrap-todo.md](content-parity-bootstrap-todo.md)

---

## Jelenlegi állapot

> Utolsó frissítés: P8.5.4 Minimal Implementation (#8)

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
