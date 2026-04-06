# Postmortem — P8.5 Content Parity Bootstrap

> Phase 8.5 lezárás: honest close, nem "minden kész".

## Eredmény

**55/55 verified parity + 18/18 endpoint smoke** — működő 5-lépéses pipeline.

| Metrika | Érték |
|---------|-------|
| Összes mező | 55 (53 ACF + 2 WP option) |
| PASS | 55 |
| FAIL | 0 |
| Endpoint image smoke | 18/18 |
| Szekciók | 10/10 lefedve |
| Pipeline | 5-lépéses orchestráció (`seed-pipeline.ps1`) |

> **Korrekció (P9.2):** Az eredeti 53/55 eredmény hibás diagnózist tartalmazott.
> Részletek a „Korrigált diagnózis" szekcióban.

## Mi számít PASS-nak

A P8.5 PASS definíciója:

1. **Működő pipeline**: `seed-pipeline.ps1` — export → import → dump → verify → endpoint smoke (5 lépés)
2. **55/55 field match**: text/CTA exact match, image structural match (URL eltérhet, alt egyezik), group kulcs-normalizálás
3. **18/18 endpoint smoke**: minden image field valid URL a live endpointban
4. **0 missing field**: minden seed.json kulcs megjelenik wp-state.json-ben
5. **Media library sync**: 18 image sideloaded (6 brand logó lokális fájlból, 12 external URL-ből)

## Mi NEM számít PASS-nak

- Navigation parity (config-driven, nem DB-ből jön)
- Vizuális regressziós teszt (screenshot diff)

---

## Korrigált diagnózis (P9.2)

Az eredeti postmortem három "maradékot" jelölt P11+ scope-ra. Mindhárom diagnózis **téves** volt:

### `bc_service_contact` + `bc_contact_info` — group sub-field kulcs prefix

**Eredeti diagnózis (TÉVES)**: „ACF field group regisztráció hiányzik"
**Valódi root cause**: A group mezők ACF regisztrációja végig megvolt (bc-service.php, bc-contact.php). A FAIL oka: `dump-acf.php` a group sub-field-eket raw ACF kulcsokkal adta vissza (`field_bc_service_contact_title`), míg a seed.json rövid kulcsokat használ (`title`). A verify-parity.ts a seed kulcsait kereste a dump-ban → `undefined`.
**Fix**: `normalize_dump_value()` seed-shape-alapú kulcs-remapping (suffix match).
**Eredmény**: 55/55 PASS.

### 6 brand logó — lokális asset sideload

**Eredeti diagnózis**: „Lokális asset, media library upload kell (P11+)"
**Valódi helyzet**: A `looks_like_image_ref()` heurisztika + `sideload_local_file()` megoldotta — a 6 brand logó a media library-be kerül (attachment #11–#16). Idempotens (CACHED re-run-ra).
**Fix**: import-seed.php bare image string detection.
**Eredmény**: Endpoint smoke PASS — mind a 6 logó valid WP media URL.

---

## Sequencing hibák

### 1. Export ownership bizonytalanság (P8.5.1 → P8.5.2)

**Mi történt**: A P8.5.1 architecture closure után a v4 plan §13.2 még `sp-infra/seed/export-seed.ts`-t írta kanonikus exportként, miközben a boundary döntés a kliens repóba helyezte.

**Hatás**: Két fájl, két repo, azonos célra — potenciális fork. Két commit kellett javításra (`235a6b1`, `38f095d`).

**Tanulság**: Boundary döntésnél az ÖSSZES hivatkozó dokumentumot azonnal frissíteni kell, nem utólag.

### 2. Round-trip shape mismatch (P8.5.4 → P8.5.5)

**Mi történt**: Az import-seed.php image `{url, alt}` object-ből URL stringet ment, a dump-acf.php URL stringet olvas vissza. A verify-parity.ts `{url, alt}` shape-et vár mindkét oldalon → garantált FAIL.

**Hatás**: Egy extra commit (`ef237c0`) kellett a dump-acf.php-hez, ahol a companion `_alt` mezőből rekonstruáljuk a shape-et.

**Tanulság**: A round-trip tesztet (export → import → dump → verify) korábban, a tooling design fázisban (P8.5.3) kellett volna végigvizsgálni, nem a verification fázisban felfedezni.

### 3. Absolute asset path (P8.5.4 → P8.5.5)

**Mi történt**: Az asset-loader.mjs Windows abszolút path-okat adott vissza (`/D:/Projects/spektra/...`). A seed.json-ben ezek nem működő URL-ek.

**Hatás**: Egy extra commit (`6d94056`) kellett a loader fix-hez + `detectLocalAssets()` warning rendszer.

**Tanulság**: Az asset resolution-t a loader implementálásánál kellett volna tesztelni — nem a verify fázisban.

### 4. WP-CLI arg interception (P8.5.5 pipeline)

**Mi történt**: A `seed-pipeline.ps1` `--dry-run`/`--verbose` flag-eket passzolt WP-CLI `eval-file`-nak, de a WP-CLI ezeket saját flag-ként kezelte → "unknown parameter" error.

**Hatás**: A PHP scriptek dual-form arg parsing-et (`--flag` + `flag`) kaptak. Nem szép, de működik.

**Tanulság**: WP-CLI `eval-file` custom arg passing-jét a tooling design-ban (P8.5.3) kellett volna specifikálni.

---

## Új standard workflow

### Seed Pipeline — egyparancsos futtatás

```powershell
# Teljes pipeline (export + import + dump + verify)
.\seed-pipeline.ps1

# Dry-run (WP-be nem ír)
.\seed-pipeline.ps1 -DryRun -Verbose

# Meglévő seed.json-nal (export kihagyás)
.\seed-pipeline.ps1 -SkipExport
```

### Runtime gate

A pipeline PASS szükséges feltétel:
- **Phase 9** adapter switch (legacy → engine) előtt
- Production cutover előtt
- Bármilyen site.ts content módosítás után (regresszió védelme)

### Fájlstruktúra

```
sp-benettcar/                         # kliens repo
└── infra/seed/
    ├── mapping.ts                    # ACF field mapping (kliens-specifikus)
    ├── export-seed.ts                # site.ts → seed.json
    ├── asset-loader.mjs              # image import stub
    └── register-asset-hooks.mjs      # loader bootstrap

sp-infra/                             # platform repo
└── seed/
    ├── import-seed.php               # seed.json → WP (ACF update_field)
    ├── dump-acf.php                  # WP → wp-state.json
    ├── verify-parity.ts              # seed.json vs wp-state.json
    ├── seed-pipeline.ps1             # orchestrátor (all-in-one)
    ├── package.json                  # verify scripts
    └── README.md                     # pipeline docs
```

### Repo felelősség

| Repo | Felel |
|------|-------|
| sp-benettcar | Content definíció (site.ts), ACF mapping, seed export |
| sp-infra | WP import, state dump, parity verification, orchestráció |
| sp-docs | Workstream definíció, postmortem, döntésnapló |

---

## P11+ scope

| Feladat | Prioritás | Blokkoló? |
|---------|-----------|-----------|
| seed-pipeline.ps1 → cross-platform (bash/CI) | Alacsony | Nem — lokális dev only |
| Automatikus ACF field group sync (site.ts → WP) | Jövő | Nem |
| Visual regression test (endpoint snapshot) | Jövő | Nem |

---

## Commit történet (15 entry)

Lásd: [content-parity-bootstrap-log.md](content-parity-bootstrap-log.md)
