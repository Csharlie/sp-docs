# Phase 11.1 — Multi-Client Boundary Audit

## Purpose

A Phase 11 nem implementációval indul, hanem audittal. Az ok egyszerű:

A BenettCar az első WordPress-backed reference consumer. A Phase 8.5 content parity, Phase 9 runtime cutover es Phase 10 stabilization lezarasa utan a platform mukodokepes — de egyetlen klienssel. A kovetkezo kockazat az implicit BenettCar coupling: ha a masodik kliens onboardolasa a BenettCar reverse-engineeringjevel kezdodik, az nem skalazodik.

Ez a dokumentum felterkepezi a jelenlegi repohatarok allapotat, es evidence-based modon ramutat arra, mi platform-owned, mi client-owned, mi infra-owned, es mi docs/governance-owned. Az eredmeny alapjan dontheto el a P11.2+ bontasa.

- BenettCar a jelenlegi reference consumer
- A kovetkezo kockazat az implicit BenettCar coupling (hardcoded section builderek, path-ok, switch-case-ek)
- A cel: reproducible multi-client onboarding, ahol az uj kliens nem fugg a BenettCar kodbazis ismeretetol

---

## Current System Baseline

A teljes runtime path a WordPress ACF mezoktol a renderelt komponensig:

```
WordPress ACF mezo ertekek (WP admin, ACF Pro)
  → sp-infra/acf/sections.php                    section builderek (ACF → JSON)
  → sp-infra/acf/media-helper.php                media normalizalas
  → sp-infra/plugin/spektra-api/includes/
      class-response-builder.php                  response osszeallitas
  → /wp-json/spektra/v1/site                      REST endpoint (JSON response)
  → sp-benettcar/src/data/create-adapter.ts       adapter factory (WP vs JSON)
  → sp-benettcar/src/data/wp-mapper.ts            raw JSON → SiteData mapping
  → sp-benettcar/src/data/normalize-site-data.ts  consumer-safety cleanup
  → sp-platform/packages/data/src/validate.ts     SiteData validacio
  → sp-platform/packages/runtime/src/context.tsx  SiteDataProvider (React context)
  → sp-platform/packages/templates/src/
      LandingTemplate.tsx                         page template
  → sp-platform/packages/runtime/src/
      section-renderer.tsx                        section dispatch
  → sp-benettcar/src/sections/bc-*/               kliens section komponensek
```

---

## Platform-Owned

Az `sp-platform` repo (`@spektra/*` csomagok) a platform core. Zero BenettCar coupling.

### SiteData contract

| | |
|---|---|
| **File** | `sp-platform/packages/types/src/site.ts` |
| **Export** | `SiteData`, `SiteMeta` |
| **Miert platform-owned** | Ez az egyetlen canonical adatszerkezet, amit minden adapter produkál és minden template fogyaszt |
| **BenettCar coupling** | Nincs |
| **Multi-client reuse** | Biztonságos |

### Media / CTA / Navigation types

| | |
|---|---|
| **Files** | `sp-platform/packages/types/src/media.ts`, `cta.ts`, `navigation.ts` |
| **Exports** | `Media`, `MediaVariant`, `MediaSource`, `CallToAction`, `NavItem`, `Navigation` |
| **Miert platform-owned** | Shared UI contract — minden kliens ugyanezeket a tipusokat hasznalja |
| **BenettCar coupling** | Nincs (media.ts-ben van egy design-explanation komment a BenettCar-rol, de runtime hatasa nincs) |
| **Multi-client reuse** | Biztonságos |

### SiteDataAdapter interface

| | |
|---|---|
| **File** | `sp-platform/packages/types/src/adapter.ts` |
| **Export** | `SiteDataAdapter` (`load`, `init?`, `revalidate?`, `onError?`) |
| **Miert platform-owned** | DI contract — minden adatforras ezt implementalja |
| **BenettCar coupling** | Nincs |
| **Multi-client reuse** | Biztonságos |

### createWordPressAdapter

| | |
|---|---|
| **File** | `sp-platform/packages/data/src/wordpress.ts` |
| **Export** | `createWordPressAdapter(config: WordPressAdapterConfig)` |
| **Miert platform-owned** | Generic WP adapter factory — `apiBase`, `endpoint`, `mapResponse` mind config-driven |
| **BenettCar coupling** | Nincs (nincs hardcoded domain vagy section type) |
| **Multi-client reuse** | Biztonságos |

### createJsonAdapter

| | |
|---|---|
| **File** | `sp-platform/packages/data/src/json-adapter.ts` |
| **Export** | `createJsonAdapter(config: JsonAdapterConfig)` |
| **Miert platform-owned** | Generic JSON adapter — URL fetch vagy inline data |
| **BenettCar coupling** | Nincs |
| **Multi-client reuse** | Biztonságos |

### validateSiteData

| | |
|---|---|
| **File** | `sp-platform/packages/data/src/validate.ts` |
| **Export** | `validateSiteData(input): SiteDataValidationResult` |
| **Miert platform-owned** | Structural SiteData validacio — nem section-data-specifikus |
| **BenettCar coupling** | Nincs |
| **Multi-client reuse** | Biztonságos |

### SiteDataProvider

| | |
|---|---|
| **File** | `sp-platform/packages/runtime/src/context.tsx` |
| **Export** | `SiteDataProvider`, `useSiteData()` |
| **Miert platform-owned** | React context/lifecycle — adapter.load() → data/loading/error state |
| **BenettCar coupling** | Nincs |
| **Multi-client reuse** | Biztonságos |

### LandingTemplate

| | |
|---|---|
| **File** | `sp-platform/packages/templates/src/LandingTemplate.tsx` |
| **Export** | `LandingTemplate` (registry, header?, footer?, fallback?, loading?, error?, pageSlug?, className?) |
| **Miert platform-owned** | Generic page shell — header/footer DI, section rendering delegalt |
| **BenettCar coupling** | Nincs (header/footer injected, nem imported) |
| **Multi-client reuse** | Biztonságos |

### SectionRenderer

| | |
|---|---|
| **File** | `sp-platform/packages/runtime/src/section-renderer.tsx` |
| **Export** | `SectionRenderer` (sections, registry, fallback?) |
| **Miert platform-owned** | Type-dispatch loop — registry.resolve(type) → component render |
| **BenettCar coupling** | Nincs |
| **Multi-client reuse** | Biztonságos |

**Platform verdict:** Az sp-platform repo teljesen kliens-agnosztikus. Zero runtime BenettCar coupling (egy komment-szintu referencia a `media.ts`-ben, lasd F6). Multi-client reuse-ra kesz.

---

## Client-Owned

Az `sp-benettcar` repo a BenettCar kliens teljes frontend + WP overlay implementacioja.

### bc-* section komponensek (10 db)

| Section | Component | Schema | Definition |
|---|---|---|---|
| bc-hero | `src/sections/bc-hero/bc-hero.component.tsx` | `bc-hero.schema.ts` | `bc-hero.definition.ts` |
| bc-brand | `src/sections/bc-brand/bc-brand.component.tsx` | `bc-brand.schema.ts` | `bc-brand.definition.ts` |
| bc-gallery | `src/sections/bc-gallery/bc-gallery.component.tsx` | `bc-gallery.schema.ts` | `bc-gallery.definition.ts` |
| bc-services | `src/sections/bc-services/bc-services.component.tsx` | `bc-services.schema.ts` | `bc-services.definition.ts` |
| bc-service | `src/sections/bc-service/bc-service.component.tsx` | `bc-service.schema.ts` | `bc-service.definition.ts` |
| bc-about | `src/sections/bc-about/bc-about.component.tsx` | `bc-about.schema.ts` | `bc-about.definition.ts` |
| bc-team | `src/sections/bc-team/bc-team.component.tsx` | `bc-team.schema.ts` | `bc-team.definition.ts` |
| bc-assistance | `src/sections/bc-assistance/bc-assistance.component.tsx` | `bc-assistance.schema.ts` | `bc-assistance.definition.ts` |
| bc-contact | `src/sections/bc-contact/bc-contact.component.tsx` | `bc-contact.schema.ts` | `bc-contact.definition.ts` |
| bc-map | `src/sections/bc-map/bc-map.component.tsx` | `bc-map.schema.ts` | `bc-map.definition.ts` |

- **Miert client-owned:** Minden section BenettCar-specifikus tartalom, vizualitast es domain szemantikat hordoz
- **Pattern extractability:** A 4-fajos minta (component + schema + definition + index) replikalhato uj kliensnel — de a tartalom mindig kliens-specifikus
- **Imports:** `@spektra/types` (Media, CallToAction), `@spektra/components` (UI primitvek), `lucide-react` (ikonok)

### Section registry

| | |
|---|---|
| **File** | `src/registry.ts` |
| **Miert client-owned** | Kliens hatarozza meg, milyen section type-ok leteznek es milyen komponens rendereli oket |
| **Imports** | `createSectionRegistry`, `registerSections` from `@spektra/runtime` |
| **Extractability** | A registry minta generic — a regisztralt definiciok kliens-specifikusak |

### wp-mapper.ts

| | |
|---|---|
| **File** | `src/data/wp-mapper.ts` |
| **Miert client-owned** | Section-aware mapping: a 10 bc-* section type switch-case-ben van, mert minden kliens mas section-oket definiál |
| **Extractability** | A helper fuggvenyek (asString, asNumber, isRecord, maybeMedia, maybeCta) extractable. A mapSectionData switch es a per-section mapperek kliens-specifikusak |
| **Kliens-specifikus** | 10 bc-* case a mapSectionData switch-ben, 10 mapBc* fuggveny |

### normalize-site-data.ts

| | |
|---|---|
| **File** | `src/data/normalize-site-data.ts` |
| **Miert client-owned** | Section-aware normalizer: per-section render-safety logika, section-specifikus dropper |
| **Extractability** | A cleanOptional, cleanCta, isRecord segédfüggvények extractable. A normalizeSectionData switch és a per-section normalizálók kliens-specifikusak |
| **Kliens-specifikus** | 10 bc-* case a switch-ben, 10 normalizeBc* függvény |

### site.ts (baseline content)

| | |
|---|---|
| **File** | `src/data/site.ts` |
| **Miert client-owned** | A BenettCar teljes tartalma JSON formátumban (mock/fallback): szövegek, navigáció, 10 section, márkalogók |
| **Extractability** | Nincs — ez 100% kliens tartalom |
| **Kliens-specifikus** | Minden: domain (`benettcar.hu`), locale (`hu`), kontakt adatok, csapat, márkanevek |

### App.tsx

| | |
|---|---|
| **File** | `src/App.tsx` |
| **Miert client-owned** | Runtime composition: adapter + registry + template + shell + branded error/loading UI összekötése |
| **Extractability** | A composition minta (Provider → Template) replicable — a shell komponensek és szövegek kliens-specifikusak |
| **Kliens-specifikus** | Magyar nyelvu error UI ("A tartalom nem elerheto"), loading UI, fallback UI, AppHeader/AppFooter |

### Shell (Header, Footer)

| | |
|---|---|
| **Files** | `src/shell/Header.tsx`, `src/shell/Footer.tsx` |
| **Miert client-owned** | Kliens branding: navigácio elrendezés, CTA célpont, footer szekciók csoportosítása |
| **Imports** | `NavigationBar`, `FooterBlock` from `@spektra/components`; `TemplateShellProps` from `@spektra/templates` |
| **Extractability** | A mapping minta generic — a szövegek és elrendezés kliens-specifikus |

### Theme

| | |
|---|---|
| **File** | `src/theme/bc-theme.ts` |
| **Miert client-owned** | BenettCar szinvilág: graphite scale, neon-blue accent, red-accent |
| **Imports** | `basePreset` from `@spektra/themes` |
| **Extractability** | A basePreset extend minta replicable — a paletta kliens-specifikus |

### Konfiguracio

| File | Tartalom | Kliens-specifikus |
|---|---|---|
| `.env` | `VITE_DATA_SOURCE=wordpress`, `VITE_WP_API_BASE=http://benettcar.local` | Domain, data source |
| `vite.config.ts` | React plugin, port 5174 | Port |
| `infra/config.php` | client_slug, allowed_origins, site_defaults, navigation, sections lista | Minden |
| `infra/acf/field-groups.php` | ACF mezo definiciok 10 bc-* section-hoz | Minden |
| `infra/seed/export-seed.ts` | site.ts → seed.json export | Kliens data path |
| `infra/seed/mapping.ts` | ACF mapping szabályok | Section-specifikus |

---

## Infra-Owned

Az `sp-infra` repo a WordPress-oldali platform infrastruktura.

### REST plugin

| | |
|---|---|
| **File** | `sp-infra/plugin/spektra-api/spektra-api.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — overlay config-based, nincs hardcoded kliens-referencia |
| **BenettCar feltetelezesek** | Nincs (kommentben van egy overlay path pelda, de runtime hatasa nincs) |
| **Template candidate** | Nem — ez a production plugin, nem kell masolni, csak deploy-olni |

### Response builder

| | |
|---|---|
| **File** | `sp-infra/plugin/spektra-api/includes/class-response-builder.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — config-driven (site_defaults, navigation, pages/sections) |
| **BenettCar feltetelezesek** | Nincs |
| **Template candidate** | Nem — production component |

### CORS handler

| | |
|---|---|
| **File** | `sp-infra/plugin/spektra-api/includes/class-cors.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — config['allowed_origins'] alapjan mukodik |
| **BenettCar feltetelezesek** | Nincs |

### REST controller

| | |
|---|---|
| **File** | `sp-infra/plugin/spektra-api/includes/class-rest-controller.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato |
| **BenettCar feltetelezesek** | Nincs |

### Section builderek — TECHNIKAI ADOSSAG

| | |
|---|---|
| **File** | `sp-infra/acf/sections.php` |
| **Ownership** | Infra-owned, DE: jelenleg BenettCar-specifikus |
| **Reusability** | NEM ujrahasznosithato as-is — 10 hardcoded `spektra_build_bc_*()` fuggveny |
| **BenettCar feltetelezesek** | Eros: `spektra_build_bc_hero()`, `spektra_build_bc_brand()`, ... mind bc-* prefixes, BenettCar section szemantikával |
| **Template candidate** | A minta (section slug → builder function) template-ezheto; a builder fuggvenyek kliens overlay-be tartoznak |

Jelenlegi builder fuggvenyek (`sp-infra/acf/sections.php`):
```
spektra_build_bc_hero()
spektra_build_bc_brand()
spektra_build_bc_gallery()
spektra_build_bc_services()
spektra_build_bc_service()
spektra_build_bc_about()
spektra_build_bc_team()
spektra_build_bc_assistance()
spektra_build_bc_contact()
spektra_build_bc_map()
```

Ez a fajl sajat magat is jelzi technikai adossagkent (15. sor: `bc-* specific — acknowledged as P11.3 technical debt`).

### Media helper

| | |
|---|---|
| **File** | `sp-infra/acf/media-helper.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — ACF image → Spektra Media shape konverzio |
| **BenettCar feltetelezesek** | Nincs |
| **Template candidate** | Nem — production utility |

### ACF helpers

| | |
|---|---|
| **File** | `sp-infra/acf/helpers.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — `spektra_get_field()` safe getter |
| **BenettCar feltetelezesek** | Nincs |

### Seed pipeline orchestrator — RESZLEGES COUPLING

| | |
|---|---|
| **File** | `sp-infra/seed/seed-pipeline.ps1` |
| **Ownership** | Infra-owned |
| **Reusability** | Reszleges — hardcoded `sp-clients\sp-benettcar` path (41. sor) |
| **BenettCar feltetelezesek** | `$ClientDir = Join-Path $SpRoot 'sp-clients\sp-benettcar'` |
| **Template candidate** | Igen, `-Client` parameter hozzaadasaval (mint `link-overlay.ps1`-ben mar megvan) |

### import-seed.php

| | |
|---|---|
| **File** | `sp-infra/seed/import-seed.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — `--client-dir` CLI parameter-driven |
| **BenettCar feltetelezesek** | Nincs |

### dump-acf.php

| | |
|---|---|
| **File** | `sp-infra/seed/dump-acf.php` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato |
| **BenettCar feltetelezesek** | Nincs |

### verify-parity.ts

| | |
|---|---|
| **File** | `sp-infra/seed/verify-parity.ts` |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — strukturalis JSON osszehasonlitas |
| **BenettCar feltetelezesek** | Nincs |

### verify-endpoint.php — RESZLEGES COUPLING

| | |
|---|---|
| **File** | `sp-infra/seed/verify-endpoint.php` |
| **Ownership** | Infra-owned |
| **Reusability** | Reszleges — hardcoded bc-* section type check-ek (88–114. sor) |
| **BenettCar feltetelezesek** | `case 'bc-hero':`, `case 'bc-brand':`, ... hardcoded assertion-ok |
| **Template candidate** | A logika reusable ha a section check-ek data-driven-re irodik at |

### Image sideload

| | |
|---|---|
| **File** | `sp-infra/seed/import-seed.php` (240–350. sor) |
| **Ownership** | Infra-owned, generic |
| **Reusability** | Teljesen ujrahasznosithato — `sideload_image()`, `sideload_external_url()`, `sideload_local_file()` |
| **BenettCar feltetelezesek** | Nincs |

### Overlay config loading

| | |
|---|---|
| **Pattern** | Symlink/Junction minta |
| **Plugin loading** | `sp-infra/plugin/spektra-api/spektra-api.php` (32–42. sor) |
| **Symlink script** | `sp-infra/scripts/link-overlay.ps1` (parameterezheto `-Client` flaggel) |
| **Reusability** | Teljesen ujrahasznosithato — a kliens overlay-t symlink-eli a WP runtime-ba |
| **BenettCar feltetelezesek** | `link-overlay.ps1` default erteke `benettcar`, de parameterezheto |

### Bootstrap / setup scriptek

| File | Cel | Parameterezheto |
|---|---|---|
| `sp-infra/scripts/link-overlay.ps1` | Kliens overlay → WP plugins junction | Igen (`-Client`) |
| `sp-infra/scripts/link-plugin.ps1` | sp-infra plugin → WP plugins symlink | Igen |
| `sp-infra/scripts/bootstrap.ps1` | Teljes bootstrap orchestrator | Ellenorizendo |
| `sp-infra/scripts/setup-env.ps1` | Environment setup | Ellenorizendo |

---

## Docs / Governance-Owned

Az `sp-docs` repo a knowledge base es governance dokumentacio.

### Parity gate

| | |
|---|---|
| **File** | `sp-docs/knowledge/guardrails/parity-gate.md` |
| **Governance rule** | CMS Runtime Parity Gate — adapter switch production cutover tiltott amig WP content nem egyezik site.ts-sel (55/55 PASS) |
| **Multi-client doc** | Igen — `repos:` mezoben `[sp-infra, sp-clients]`, globalis ervenyu |
| **Hianyzo multi-client doc** | Nincs definiálva, hogy uj kliensnel hany field match szukseges (55 BenettCar-specifikus szam) |

### No silent fallback

| | |
|---|---|
| **File** | `sp-docs/knowledge/guardrails/no-silent-fallback.md` |
| **Governance rule** | WP adapter throw on non-2xx, soha nem fallback JSON-ra; explicit branded error UI |
| **Multi-client doc** | Igen — globalis ervenyu |
| **Hianyzo multi-client doc** | Nincs — ez mar generic |

### Content parity bootstrap

| | |
|---|---|
| **File** | `sp-docs/content-parity-bootstrap.md` |
| **Governance rule** | Phase 8.5 workstream definicio: ACF field mapping tablak, seed.json shape, parity definicio |
| **Multi-client doc** | A minta ujrahasznosithato, de a dokumentum BenettCar-specifikus field tablakat tartalmaz |
| **Hianyzo multi-client doc** | Generic content parity bootstrap template kellene |

### SPK knowledge struktura

| | |
|---|---|
| **Files** | `sp-docs/.github/agents/SPK.agent.md`, `sp-docs/knowledge/CHANGELOG.md` |
| **Governance rule** | SPK v1.1.0 — canonical typed dirs, phase index, frontmatter konvenciok |
| **Multi-client doc** | Struktura generic — de az osszes jelenlegi knowledge doc BenettCar kontextusbol szuletett |

### Phase index-ek

| File | Phase |
|---|---|
| `sp-docs/knowledge/phases/phase-8.1.md` | WordPress Mapper Boundary origin |
| `sp-docs/knowledge/phases/phase-8.5.md` | Content parity origin |
| `sp-docs/knowledge/phases/phase-9.1.md` | Runtime cutover origin |
| `sp-docs/knowledge/phases/p9.1-runtime-cutover-baseline.md` | Phase-specific gate |

### Hianyzo governance dokumentacio

- Uj kliens onboarding workflow dokumentum
- Generic overlay template guide
- Multi-client CORS konfiguracio guide
- Section builder delegation pattern (infra → overlay)
- Kliens slug konvencio es naming szabalyok

---

## Boundary Matrix

| Concern | Platform | Infra | Client | Docs | Notes |
|---|---|---|---|---|---|
| SiteData contract | `@spektra/types` | — | Fogyasztja | — | Tiszta, generic |
| Section rendering | `SectionRenderer` | — | Registry + components | — | Tiszta delegation |
| Section implementation | — | — | `src/sections/bc-*/**` | — | 100% kliens |
| WordPress REST endpoint | — | `plugin/spektra-api/` | — | — | Generic, config-driven |
| ACF field registration | — | Loader (`spektra-api.php`) | `infra/acf/field-groups.php` | — | Kliens overlay |
| ACF value extraction | — | `acf/helpers.php` (generic) | — | — | Tiszta |
| Section building (ACF → JSON) | — | `acf/sections.php` **!** | `infra/config.php` (section list) | — | **P1: bc-* hardcoded az infra-ban** |
| Seed export | — | — | `infra/seed/export-seed.ts` | — | Kliens adatbol |
| Seed import | — | `seed/import-seed.php` | — | — | Generic |
| Media sideload | — | `seed/import-seed.php` | — | — | Generic |
| Mapper | — | — | `src/data/wp-mapper.ts` | — | Section-aware, kliens-specifikus |
| Normalizer | — | — | `src/data/normalize-site-data.ts` | — | Section-aware, kliens-specifikus |
| Runtime adapter | `createWordPressAdapter` | — | `src/data/create-adapter.ts` | — | Factory generic, config kliens |
| Runtime error UI | `LandingTemplate` (error slot) | — | `src/App.tsx` (branded UI) | — | Slot generic, content kliens |
| Parity gate | — | `seed/verify-parity.ts` | — | `guardrails/parity-gate.md` | Generic |
| CORS | — | `class-cors.php` | `infra/config.php` (origins) | — | Config-driven |
| New client onboarding | — | — | — | **Nincs** | **P1: nem letezik dokumentum** |

---

## Extraction Candidates

### 1. wp-mapper.ts helper fuggvenyek

| | |
|---|---|
| **Jelenlegi hely** | `sp-benettcar/src/data/wp-mapper.ts` |
| **Javasolt owner** | `sp-platform/packages/data` (uj modul: `wp-mapper-helpers`) |
| **Ok** | `isRecord`, `asString`, `asNumber`, `asBoolean`, `asArray`, `maybeMedia`, `maybeCta`, `optionalString` — mind generic, SiteData typusokra epulnek |
| **Korai extraction kockazat** | Alacsony — a fuggvenyek stabilak, nincsenek section-specifikus fuggosegek |

### 2. normalize-site-data.ts utility fuggvenyek

| | |
|---|---|
| **Jelenlegi hely** | `sp-benettcar/src/data/normalize-site-data.ts` |
| **Javasolt owner** | `sp-platform/packages/data` (uj modul: `normalizer-helpers`) |
| **Ok** | `cleanOptional`, `cleanCta`, `isRecord`, `asTypedArray`, `isDefined` — SiteData contract-szintu utility-k |
| **Korai extraction kockazat** | Alacsony |

### 3. Section builder dispatch (sections.php)

| | |
|---|---|
| **Jelenlegi hely** | `sp-infra/acf/sections.php` |
| **Javasolt owner** | Ket resz: (1) generic dispatch → sp-infra, (2) bc-* builderek → `sp-benettcar/infra/acf/sections/` overlay |
| **Ok** | 10 hardcoded `spektra_build_bc_*()` fuggveny blokkolja a masodik klienst |
| **Korai extraction kockazat** | Kozepesen alacsony — a dispatch pattern egyszeru, a builderek mar izoláltanak tunoek |

### 4. seed-pipeline.ps1 parameterezes

| | |
|---|---|
| **Jelenlegi hely** | `sp-infra/seed/seed-pipeline.ps1` |
| **Javasolt owner** | Marad sp-infra, de `-Client` parameter szukseges |
| **Ok** | Hardcoded `sp-clients\sp-benettcar` path |
| **Korai extraction kockazat** | Alacsony — egyszeru parameter csere |

### 5. verify-endpoint.php generikussa tetele

| | |
|---|---|
| **Jelenlegi hely** | `sp-infra/seed/verify-endpoint.php` |
| **Javasolt owner** | Marad sp-infra, de data-driven section check kell |
| **Ok** | Hardcoded bc-* section type assertion-ok |
| **Korai extraction kockazat** | Alacsony |

### 6. create-adapter.ts minta

| | |
|---|---|
| **Jelenlegi hely** | `sp-benettcar/src/data/create-adapter.ts` |
| **Javasolt owner** | Marad kliens — de a minta dokumentalando mint reference pattern |
| **Ok** | A WP/JSON valasztas minta minden kliens adapter factory-jahoz szukseges |
| **Korai extraction kockazat** | Nincs — ez dokumentacio kerdes, nem code move |

---

## Non-Extractable BenettCar Specifics

Az alabbi elemek **kizarolag BenettCar-specifikusak** es soha nem kerulhetnek platformba vagy infra-ba:

- **Section type nevek:** `bc-hero`, `bc-brand`, `bc-gallery`, `bc-services`, `bc-service`, `bc-about`, `bc-team`, `bc-assistance`, `bc-contact`, `bc-map`
- **Tartalom:** Minden szoveg a `site.ts`-ben (cegnev, leiras, csapattagok, kontakt adatok, szolgaltatasok)
- **Vizualis dontesek:** bc-theme.ts szinpaletta (graphite, neon-blue, red-accent), section-specifikus layout-ok
- **Sorrend:** A 10 section sorrendje es a page slug (`fooldal`)
- **Domain szemantika:** Autokereskedelem-specifikus: maraklogok, VW Konszern brandek, szerviz lista, segélyszolgálat
- **Brand assetek:** `src/assets/brands/` (vw-logo, audi-logo, skoda-logo, porsche-logo, lamborghini-logo, bugatti-logo)
- **Navigacio:** Magyar nyelvu cimkek es href celpont struktura
- **Kontakt adatok:** Telefonszamok, email cimek, fizikai cim

---

## Bootstrap Requirements For A New Client

A BenettCar reference implementacio alapjan az uj kliens onboardolasahoz az alabbi elemek szuksegesek (megfigyelt, nem eloirt lista):

### Frontend (uj sp-clients/sp-{slug}/ repo)

| # | Requirement | BenettCar pelda | Megjegyzes |
|---|---|---|---|
| 1 | Client slug | `benettcar` | Konvencio: lowercase, kotojelbol-mentes |
| 2 | Section lista + types | 10 db `bc-*` | Prefix = kliens slug rovide (pl. `bc-` = benettcar) |
| 3 | Section schemak | `src/sections/bc-*/bc-*.schema.ts` | TypeScript interface per section |
| 4 | Section komponensek | `src/sections/bc-*/bc-*.component.tsx` | React komponens per section |
| 5 | Section definiciok | `src/sections/bc-*/bc-*.definition.ts` | `SectionDefinition<T>` per section |
| 6 | Section registry | `src/registry.ts` | `createSectionRegistry()` + `registerSections()` |
| 7 | Baseline content (site.ts) | `src/data/site.ts` | JSON mode fallback + seed pipeline input |
| 8 | WP mapper | `src/data/wp-mapper.ts` | Section-aware raw JSON → SiteData |
| 9 | Normalizer | `src/data/normalize-site-data.ts` | Section-aware render-safety cleanup |
| 10 | Adapter factory | `src/data/create-adapter.ts` | WP vs JSON routing |
| 11 | App.tsx | `src/App.tsx` | Provider + Template + shells + error/loading UI |
| 12 | Shell (Header/Footer) | `src/shell/Header.tsx`, `Footer.tsx` | Branded navigation + footer |
| 13 | Theme | `src/theme/bc-theme.ts` | Tailwind preset a kliens szinpalettajaval |
| 14 | .env | `.env` | `VITE_DATA_SOURCE`, `VITE_WP_API_BASE` |
| 15 | Vite config | `vite.config.ts` | Port, React plugin |
| 16 | Tesztek | `src/data/*.test.ts` | Mapper + normalizer unit tesztek |

### WordPress infra overlay (sp-clients/sp-{slug}/infra/)

| # | Requirement | BenettCar pelda |
|---|---|---|
| 17 | config.php | `infra/config.php` — slug, origins, site_defaults, navigation, section lista |
| 18 | ACF field groups | `infra/acf/field-groups.php` — `acf_add_local_field_group()` per section |
| 19 | Seed export | `infra/seed/export-seed.ts` — site.ts → seed.json |
| 20 | Seed mapping | `infra/seed/mapping.ts` — ACF field → SiteData struct |

### Infra (sp-infra — mar letezik, de modositando)

| # | Requirement | Jelenlegi allapot |
|---|---|---|
| 21 | Section builderek delegalas | **P1: sections.php bc-* hardcoded** — overlay-ba kell delegalni |
| 22 | seed-pipeline.ps1 parameterezes | **P2: hardcoded benettcar path** — `-Client` parameter kell |
| 23 | verify-endpoint.php generalas | **P2: hardcoded bc-* check-ek** — data-driven kell |
| 24 | Overlay symlink | `link-overlay.ps1 -Client {slug}` — mar parameterezheto |
| 25 | WP runtime setup | `.local/wp-runtimes/{slug}/` — kezileg kell letrehozni |

### Runtime konfiguracio

| # | Requirement |
|---|---|
| 26 | CORS allowed_origins a kliens overlay config.php-ban |
| 27 | WP site/page letrehozasa a kliens tartalommal |
| 28 | ACF field-ok kitoltese (manual vagy seed import) |
| 29 | Parity verification (verify-parity.ts PASS) |

---

## Findings

### P1 — Masodik kliens onboardolast blokkolo

**F1: Section builderek hardcoded az sp-infra-ban**
| | |
|---|---|
| **Severity** | P1 |
| **Repo/file** | `sp-infra/acf/sections.php` |
| **Miert szamit** | 10 `spektra_build_bc_*()` fuggveny — ha masodik kliens jon, az infra-ban is bc-* builderek futnak, vagy kell egy uj dispatch ág, ami a jelenlegi switch-et tovabb boviti kliens-specifikus kodddal egy shared repo-ban |
| **Follow-up** | P11 kovetkezo lepes: section builder delegation pattern — generic dispatch az infra-ban, kliens-specifikus builderek az overlay-ban |

**F2: Uj kliens onboarding dokumentacio nem letezik**
| | |
|---|---|
| **Severity** | P1 |
| **Repo/file** | `sp-docs/` — nincs teljes, standalone, reprodukalhato new-client-guide.md (reszleges anyag letezik: `content-parity-bootstrap.md`, v4 Phase 11 leiras) |
| **Miert szamit** | A BenettCar setup teljes ismerete jelenleg reszben implicit — egyes lepesek dokumentaltak, de nincs egyetlen vegigjarhato onboarding workflow |
| **Follow-up** | P11 kovetkezo lepes: onboarding guide |

### P2 — Nem blokkol, de manualis munkat/drift kockazatot teremt

**F3: seed-pipeline.ps1 hardcoded benettcar path**
| | |
|---|---|
| **Severity** | P2 |
| **Repo/file** | `sp-infra/seed/seed-pipeline.ps1` (41. sor) |
| **Miert szamit** | Masodik kliens seed pipeline-jahoz vagy masolni kell a scriptet, vagy meg kell parameterezheto |
| **Follow-up** | `-Client` parameter hozzaadasa (mint `link-overlay.ps1`-ben) |

**F4: verify-endpoint.php hardcoded bc-* section check-ek**
| | |
|---|---|
| **Severity** | P2 |
| **Repo/file** | `sp-infra/seed/verify-endpoint.php` (88–114. sor) |
| **Miert szamit** | Masodik kliens endpoint verifikaciojahoz a section check-eket kezileg kell atirni |
| **Follow-up** | Data-driven section check (config-bol vagy response-bol kinyerve) |

**F5: wp-mapper.ts es normalize-site-data.ts helper fuggvenyek duplikacio kockazata**
| | |
|---|---|
| **Severity** | P2 |
| **Repo/file** | `sp-benettcar/src/data/wp-mapper.ts`, `normalize-site-data.ts` |
| **Miert szamit** | A helper fuggvenyek (isRecord, asString, maybeMedia, cleanOptional, stb.) minden uj kliens mappereben es normalizer-eben ujra lesznek irva |
| **Follow-up** | Extraction review a P11 kesobbi szakaszaban |

### P3 — Dokumentacio, tisogatas

**F6: media.ts design-explanation komment**
| | |
|---|---|
| **Severity** | P3 |
| **Repo/file** | `sp-platform/packages/types/src/media.ts` (3. sor) |
| **Miert szamit** | Kommentben BenettCar-referenciat tartalmaz — nem runtime problema, de kozmetikai |
| **Follow-up** | Komment frissites |

**F7: Content parity bootstrap nincs template-ezve**
| | |
|---|---|
| **Severity** | P3 |
| **Repo/file** | `sp-docs/content-parity-bootstrap.md` |
| **Miert szamit** | A dokumentum BenettCar-specifikus ACF field tablakat tartalmaz — uj kliensnel a mintat kell kovetni, nem a tablakat |
| **Follow-up** | Generic content parity template |

**F8: Parity gate 55/55 field count BenettCar-specifikus**
| | |
|---|---|
| **Severity** | P3 |
| **Repo/file** | `sp-docs/knowledge/guardrails/parity-gate.md` |
| **Miert szamit** | A gate logic generic, de a "55 mezo" szam BenettCar-specifikus — uj kliensnel mas szam lesz |
| **Follow-up** | Pontositas: "N/N PASS" formulazas |

---

## sp-modules Status

Az `sp-modules` konyvtar nem letezik a `D:\Projects\spektra\` workspace-ben. A jelenlegi architektura az `sp-platform/packages/` monorepo-strukturat hasznalja (`@spektra/types`, `@spektra/data`, `@spektra/runtime`, `@spektra/templates`, `@spektra/themes`, `@spektra/components`, `@spektra/sections`). Az sp-modules nem resztvevo repo a Phase 11-ben.

---

## Recommended Next Steps

Az alabbi bontast az audit eredmenye alapjan javasolom. Ez nem alkalmazott roadmap-allapot — a meglevo `wp-integration-plan-v4.md` Phase 11 struktura marad ervenyben amig kulon dontessel nem modosul. Ha barki innen dolgozik: a v4 roadmap az autoritativ forras, nem ez a javaslat.

### P11.2 — Section Builder Delegation

| | |
|---|---|
| **Cel** | sections.php bc-* hardcode feloldasa: generic dispatch az infra-ban, kliens builderek az overlay-ban |
| **Input** | F1 finding + jelenlegi sections.php + overlay config pattern |
| **Output** | sp-infra/acf/sections.php refactored: overlay-bol tolti a kliens buildereket |
| **Done condition** | BenettCar mukodik az uj dispatch-csel; masodik kliens builder-jei overlay-bol betolthetok anelkul, hogy sp-infra modosulna |

### P11.3 — Overlay Template

| | |
|---|---|
| **Cel** | Masolhato kliens overlay template (config.php, acf/, seed/) |
| **Input** | BenettCar/infra/ mint reference + P11.2 uj dispatch pattern |
| **Output** | Template konyvtar vagy generator, amivel uj kliens overlay 30 perc alatt osszeallithato |
| **Done condition** | Egy uj dummy kliens overlay letrehozva template-bol, sections.php overlay-bol betoltve |

### P11.4 — Infra Script Parameterization

| | |
|---|---|
| **Cel** | seed-pipeline.ps1 + verify-endpoint.php generikussa tetele |
| **Input** | F3, F4 findings |
| **Output** | Mindket script `-Client` parameterrel mukodik |
| **Done condition** | `seed-pipeline.ps1 -Client benettcar` es `seed-pipeline.ps1 -Client {dummy}` egyarant lefut |

### P11.5 — New Client Onboarding Guide

| | |
|---|---|
| **Cel** | Step-by-step dokumentacio uj kliens onboardolasahoz |
| **Input** | Bootstrap Requirements lista (fent) + P11.2–P11.4 eredmenyek |
| **Output** | `sp-docs/` onboarding guide |
| **Done condition** | Egy uj kliens a guide alapjan onboard-olhato anelkul, hogy a BenettCar repot kellene tanulmanyozni |

---

## Done State

Phase 11.1 complete when:

- [x] Boundary matrix letezik es kitoltott
- [x] Ownership explicit minden concern-re, file path-okkal alatamasztva
- [x] Extraction candidates listazva kockazattal
- [x] BenettCar-specifikus elemek elkülönítve
- [x] Bootstrap requirements a BenettCar reference alapjan deriválva
- [x] Recommended next steps dokumentálva (nem alkalmazott roadmap state)
- [x] sp-modules statusza tisztázva
