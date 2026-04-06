# Postmortem — P8.5 Content Parity Bootstrap

> Phase 8.5 lezárás: honest close, nem "minden kész".

## Eredmény

**53/55 verified parity** — működő pipeline, tudatos maradékokkal.

| Metrika | Érték |
|---------|-------|
| Összes mező | 55 (53 ACF + 2 WP option) |
| PASS | 53 |
| FAIL | 2 (group field — ACF regisztráció hiányzik WP-ben) |
| Szekciók | 10/10 lefedve |
| Pipeline | egyparancsos orchestráció (`seed-pipeline.ps1`) |

## Mi számít PASS-nak

A P8.5 PASS definíciója:

1. **Működő pipeline**: `seed-pipeline.ps1` — export → import → dump → verify egyetlen paranccsal
2. **53/55 field match**: text/CTA exact match, image structural match (URL eltérhet, alt egyezik)
3. **0 missing field**: minden seed.json kulcs megjelenik wp-state.json-ben
4. **Explicit elfogadott maradékok**: a 2 FAIL + 6 lokális asset tudatosan P11+ scope-ba tolva

## Mi NEM számít PASS-nak

- Full 55/55 match
- Media library sync (image feltöltés + attachment ID)
- ACF field group regisztráció automatizálás
- Navigation parity (config-driven, nem DB-ből jön)

---

## FAIL mezők és root cause

### `bc_service_contact` — group (8 sub-field)

**Tünet**: `update_field()` false-t ad vissza, dump undefined-ot olvas.
**Root cause**: A `bc_service_contact` group field nincs regisztrálva az ACF plugin-ben (nincs field group definition a WP admin-ban). Az `update_field()` ACF regisztrált field-et vár — regisztrálatlan key-re false-t ad.
**Fix (P11+)**: ACF field group JSON export létrehozása, ami tartalmazza a contact group definíciót.

### `bc_contact_info` — group (3 sub-field)

**Tünet**: Azonos — `update_field()` false, dump undefined.
**Root cause**: Azonos — nincs ACF field group regisztráció.
**Fix (P11+)**: Ugyanaz mint fent.

### 6 brand logó — lokális asset referencia

**Tünet**: Seed-ben `src/assets/brands/vw-logo.jpg` (projekt-relatív path), nem WP media URL.
**Root cause**: A site.ts ES importokat használ (`import vwLogo from '...jpg'`), amiket Vite build-ben URL-re old. Seed export kontextusban nincs Vite → asset-loader stub-olja fájlnévre.
**Fix (P11+)**: Media library upload pipeline — lokális képek feltöltése WP-be, attachment URL visszaírása.

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

## P11+ scope — tudatos maradékok

| Feladat | Prioritás | Blokkoló? |
|---------|-----------|-----------|
| ACF field group JSON regisztráció (bc_service_contact, bc_contact_info) | Közepes | Nem — 53/55 PASS |
| Media library upload pipeline (brand logók) | Alacsony | Nem — placeholder path elfogadott |
| seed-pipeline.ps1 → cross-platform (bash/CI) | Alacsony | Nem — lokális dev only |
| Automatikus ACF field group sync (site.ts → WP) | Jövő | Nem |

---

## Commit történet (15 entry)

Lásd: [content-parity-bootstrap-log.md](content-parity-bootstrap-log.md)
