---
title: Phase 14 — BenettCar Handover Release Hardening
status: active
type: phase-roadmap
canonical: true
created: 2026-04-25
last_updated: 2026-05-14 (P14.6 contact-form-done)
---

# Phase 14 — BenettCar Handover Release Hardening

## Purpose

A BenettCar első teljes körű ügyfél-átadás lezárása. Ez nem platform hardening fázis — elsődleges cél az átadható ügyfélállapot elérése, nem architektúra fejlesztés.

---

## Ad-hoc módosítások kezelése

A phase végrehajtása közben felmerülő spontán módosításokat az alábbi séma alapján kezeljük:

| Módosítás típusa | Kezelés |
|---|---|
| Frontend / tartalom (P14.2 témakör) | Tracker-be kerül új tételként, közösen jóváhagyva |
| Már meglévő phase-be illő technikai feladat | Az adott phase checklistjéhez adjuk hozzá ebben a doc-ban |
| Teljesen új scope | `P14.x` amendment-ként kerül a doc-ba, explicit döntéssel |
| Apró spontán javítás, nem blokkoló | Saját commitba kerül, P14.9-ben dokumentálva visszamenőleg |

Semmi nem veszik el: ha valami éppen nem illik sehova, elég annyi, hogy jelzed — besorolásra kerül.

---

## Kontextus

### Előzmény

P13.x sorozat (P13.1–P13.5) lezárta:
- Slot-based ACF Free admin modell (bc-services, bc-about, bc-service, bc-gallery, bc-brand, bc-team)
- Seed mapping alignment (194-mező export validálva)
- Admin UX fejlesztések: magyar labelek, accordion preview, collapsed metaboxok, WP menu support

### Jelenlegi blokkerek

**Hard Blockers** (átadás ezek nélkül nem javasolt):

1. Ügyfél frontend visszajelzések nincsenek beépítve — **feedback lista még nem érkezett be**
2. Admin guide nincs commitolva / nincs remote-on igazolható
3. Contact form fake success állapotban marad döntés nélkül
4. WordPress tartalom nincs újraszinkronizálva a módosított JSON baseline-ból
5. Staging nincs validálva
6. CORS nincs staging/production originre állítva
7. Media/local asset kockázat nincs lezárva (brand logók `src/assets/brands/` alatt)
8. Mobile/desktop smoke test nincs lefuttatva
9. Release checklist nincs dokumentálva

**Soft Blockers** (nem akadályozzák az első átadást, de dokumentálandók):

- CPT alapú gallery/services/team management
- Analytics/GTM
- SEO mély optimalizálás
- Form backend implementáció részletei (Option A választva — scope és technológia P14.6-ban)
- Scaffold/generator
- Phase 12 platform hardening folytatása

---

## Phase Map

| Phase | Cím | Státusz |
|-------|-----|---------|
| P14.0 | Remote Baseline Verification | PENDING |
| P14.1 | JSON Source Frontend Editing Mode | PENDING |
| P14.2 | Client Feedback Frontend Implementation | ✅ DONE |
| P14.3 | Content & Media Freeze | ✅ DONE (ügyfél fotók/email pending — nem blokkoló) |
| P14.4 | WordPress Resync / Seed Rehydration | ✅ DONE lokálisan (parity 196/196) + ✅ éles Rackforest WP-n (P14.7 keretében: brand logók + seed importálva, REST `200 OK`, alt parity) |
| P14.5 | Admin Handover Package | 🟡 IN PROGRESS — kézikönyv + gyors útmutató .doc committed; contact-rész screenshotok pótolandók. Admin UX: „Főoldal" menüpont ✅ + branded CMS gateway ✅ (wp.benettcar.hu root) |
| P14.6 | Contact Form Decision | ✅ DONE (2026-05-14) — `FormHandler` driver-absztrakció (`@spektra/data` pure async + `@spektra/runtime` React context), CF7 production driver, 33 unit teszt zöld, production smoke PASS (email kézbesítve). Tag: `p14.6-contact-form-done`. |
| P14.7 | Staging / Production Readiness | 🟡 IN PROGRESS (2026-05-14) — Rackforest deployment live: `wp.benettcar.hu` + `benettcar.hu/v2/` cookie-gated. SMTP ✅ (`info@benettcar.hu`). Hátralévő: auth gate disable átadáskor. Tag: `p14.7-stage-ready`. |
| P14.8 | Final Smoke Test & Release Freeze | ✅ DONE (2026-05-14) — build/test PASS, REST OK, contact form OK, browser smoke OK. Tag: `p14.8-release-freeze`. |
| P14.9 | Documentation Sync Across Repos | 🟡 IN PROGRESS |
| P14.10 | Non-BenettCar Follow-up Cleanup | DEFERRED |

---

## Fontos korlátok (az egész fázisra)

- Platform repo nem kap BenettCar-specifikus módosítást
- WordPress/ACF logika nem kerül platformba
- Client-specific builder marad `sp-benettcar/infra/acf`
- Infra core marad generic
- SiteData contract csak blocker esetén módosulhat
- CPT migráció nem indul átadás előtt
- JSON source csak frontend editing baseline, nem új architektúra
- WordPress source lesz az átadási CMS runtime
- Seed pipeline marad a JSON/site.ts → WP sync út

---

## P14.0 — Remote Baseline Verification

### Cél

Kiindulási remote és lokális állapot rögzítése, hogy a további munka kontrollált legyen.

### Elvégzendő

**Minden repóban:**
```
git status --short
git branch --show-current
git log --oneline -5
git rev-list --left-right --count origin/main...HEAD
```

**Baseline commit hash-ek rögzítése:**
- `spektra` (sp-platform)
- `sp-benettcar`
- `sp-infra`
- `sp-docs`

**Lokális ellenőrzés:**
- Nincs uncommitted módosítás `bc-services.component.tsx`-ben
- Nincs Word lock fájl (`~$*.doc`, `~$*.docx`)
- Admin guide fájlok helyzete:
  - `sp-benettcar/docs/benettcar-wordpress-admin-kezikonyv.doc` — **lokálisan létezik, nincs commitolva**
  - `sp-benettcar/docs/benettcar-wordpress-admin-gyors-utmutato.doc` — **lokálisan létezik, nincs commitolva**
  - Dönteni kell a commitolásról P14.5 előtt

**`.gitignore` javítás** (`sp-benettcar/.gitignore`):
```
~$*.doc
~$*.docx
```

### Done State

- Minden repo clean vagy ismert módosításként dokumentált
- Baseline commit hash-ek rögzítve
- `.gitignore` Word lock pattern hozzáadva
- Admin guide fájlok helyzete dokumentált

### Commit javaslat
```
chore: P14.0 baseline cleanup and gitignore handover artifacts
```

---

## P14.1 — JSON Source Frontend Editing Mode

### Cél

A BenettCar frontend WordPress nélkül, stabil JSON source-ból szerkeszthető legyen.

### Döntés

```
VITE_DATA_SOURCE=json
```

**Fontos:** `.env.frontend-editing` NEM fog működni — Vite csak `.env`, `.env.local`, `.env.[mode]` és `.env.[mode].local` fájlokat tölt be automatikusan. Kizárólag `.env.local` használandó.

### `create-adapter.ts` állapota

Már rendben van (`sp-benettcar/src/data/create-adapter.ts`):
- `json` source: `createJsonAdapter({ data: siteData })`
- `wordpress` source: `createWordPressAdapter(...)` — igényli `VITE_WP_API_BASE`-t
- Ismeretlen source: fallback `json`-ra, console warn

### Elvégzendő

1. `.env.local` létrehozása (gitignorált):
   ```
   VITE_DATA_SOURCE=json
   ```
2. A committed `.env` marad változatlanul: `VITE_DATA_SOURCE=wordpress` + `VITE_WP_API_BASE=http://benettcar.local`
3. Build/test baseline futtatása JSON mode-ban:
   ```
   npm run build
   npm test -- --run
   ```
4. Source switching dokumentálása (lásd alább)

### Source Switching Workflow

| Mode | Fájl | Tartalom | Mikor |
|------|------|----------|-------|
| Frontend editing | `.env.local` | `VITE_DATA_SOURCE=json` | P14.1–P14.3 |
| WP runtime validation | `.env.local` | `VITE_DATA_SOURCE=wordpress` + `VITE_WP_API_BASE=http://benettcar.local` | P14.4 |
| Staging | env a deploy platformon | `VITE_DATA_SOURCE=wordpress` + staging WP URL | P14.7 |

### Done State

- BenettCar frontend JSON source módban fut
- Build/test baseline PASS JSON mode-ban
- `.env.local` gitignorált, nem kerül commitba
- Source switching workflow dokumentált

### Commit javaslat
```
chore: P14.1 document JSON source frontend editing workflow
```

---

## P14.2 — Client Feedback Frontend Implementation

### Cél

Az ügyfél által adott landing page módosítások beépítése JSON source módban, külön promptokban / fejlesztési körökben.

### Fontos döntés — Tracker-alapú workflow

A módosítások **külön promptokban** érkeznek — minden egyes változtatás önálló fejlesztési kör. Ezért:

1. **P14.2 indulásakor** létre kell hozni: `sp-benettcar/docs/bc-client-feedback-tracker.md`
2. A trackerben előre fel kell sorolni az összes kért módosítást (közösen átbeszélve)
3. Minden prompt végén az agent bepipálja az elvégzett tételt
4. **P14.3 Content Freeze triggere: tracker 100%-os**

### Tracker formátum

```markdown
# BenettCar — Client Feedback Tracker

## Státusz
- Utolsó frissítés: YYYY-MM-DD
- Elkészült: X / N

## Módosítások

| # | Kategória | Leírás | Státusz | Commit |
|---|-----------|--------|---------|--------|
| 1 | content-only | ... | [ ] | — |
...
```

### Baseline lock (P14.2 indítása előtt)

Az első módosítás előtt rögzíteni kell:
- `git status` clean
- JSON mode aktív (`.env.local` tartalmazza `VITE_DATA_SOURCE=json`)
- `npm run build` PASS
- `npm test -- --run` PASS

### Változás kategóriák

| Kategória | Leírás | Érintett fájlok |
|-----------|--------|-----------------|
| content-only | szöveg, telefonszám, cím, CTA label | `site.ts` |
| media-only | kép, logó, gallery csere, alt text | `site.ts`, `src/assets/` |
| frontend layout | spacing, section order, responsive, typography | komponens `.tsx` fájlok |
| component logic | CTA behavior, anchor, form, map | komponens `.tsx` fájlok |
| CMS/admin impact | új ACF mező, label módosítás | `infra/acf/`, admin docs |
| platform impact | csak valódi reusable contract issue esetén | `sp-platform/` |

### Commit stratégia

Minden **jóváhagyott és kész** módosítás saját commitot kap:
```
feat(bc): [rövid leírás] — P14.2
```

Nem gyűjtjük össze freeze-ig — minden elvégzett módosítás azonnal commitolódik.

### Minden prompt végén elvégzendő

```
npm run build
npm test -- --run
```
Lint opcionálisan:
```
npm run lint
```

### Guardrail

- Platform core nem módosul BenettCar-specifikus feedback miatt
- SiteData contract nem módosul, ha nem muszáj
- BenettCar logika nem kerül infra core-ba
- CPT refactor nem indul

### Done State

- Tracker 100%-os
- Minden módosítás commitolva
- Build/test PASS
- Mobil/desktop vizuálisan rendben

### Commit javaslat
```
feat: P14.2 apply BenettCar client frontend feedback
```

---

## P14.3 — Content & Media Freeze

### Cél

A végleges JSON/site.ts tartalom és media source legyen az átadási baseline.

### Kritikus kockázat — brand logók

A `site.ts` jelenleg lokális asset importokat tartalmaz:
```ts
import vwLogo from '../assets/brands/vw-logo.jpg'
import audiLogo from '../assets/brands/audi-logo.png'
// ...
```
Ezek seed exportnál `non-URL` warningot generálnak. **Minden brand logóhoz explicit döntés szükséges és rögzítendő a Done State-hez.**

### Elvégzendő — Content checklist

- [ ] hero szöveg, kép
- [ ] brand logók — **minden logóhoz: WP Media Library URL vagy intentional static asset**
- [ ] gallery képek
- [ ] service kártyák
- [ ] service detail section
- [ ] about content
- [ ] team members
- [ ] assistance blokk
- [ ] contact adatok (telefon, cím, email, nyitvatartás)
- [ ] map query
- [ ] navigation (primary + footer)
- [ ] footer
- [ ] CTA-k
- [ ] SEO alapadatok (site.name, description, url, locale)

### Minden képnél döntendő

| Kép | Típus | Forrás | Döntés |
|-----|-------|--------|--------|
| (feltöltendő) | brand logó | `src/assets/brands/` | WP Media Library / static asset |

### Done State

- Tracker 100%-os ✓
- Nincs placeholder tartalom
- Minden brand logóhoz **explicit URL döntés rögzítve**
- Nincs dokumentálatlan local-only path
- Alt text minden képen
- Végleges tartalom JSON source-ban kész
- `npm run seed:export:dry` lefutott, warning lista ismert és dokumentált

### Commit javaslat
```
chore: P14.3 finalize BenettCar content and media baseline
```

---

## P14.4 — WordPress Resync / Seed Rehydration

### Cél

A módosított JSON/site.ts tartalom visszajuttatása WordPressbe, hogy a WP-backed verzió ugyanazt a végleges tartalmat szolgálja ki.

### Workflow

```
1. site.ts / JSON source végleges (P14.3 DONE)
2. npm run seed:export:dry  ← warning-ok ellenőrzése
3. npm run seed:export      ← seed.json előállítása
4. seed import WP runtime ellen
5. /wp-json/spektra/v1/site ellenőrzése
6. VITE_DATA_SOURCE=wordpress frontend visszaállítása
7. JSON render vs WP render parity ellenőrzés
```

### Elvégzendő

**Seed export:**
```
npm run seed:export:dry
npm run seed:export
```

Ellenőrzés:
- Mezőszám (P13.4 óta: 194 mező)
- Local asset warning lista
- Seed output path

**WP import:**
```
# dry-run először
# majd real import
```

**Endpoint:**
```
GET /wp-json/spektra/v1/site
```

**Frontend visszaállítás** (`.env.local`):
```
VITE_DATA_SOURCE=wordpress
VITE_WP_API_BASE=http://benettcar.local
```

**Parity check:** JSON mode render ≈ WP mode render

**Eltérés esetén vizsgálandó:**
- `infra/seed/mapping.ts`
- `infra/acf/` field groups
- PHP builders (`sp-benettcar` overlay)
- `src/data/wp-mapper.ts`
- `src/data/normalize-site-data.ts`
- media resolver

### Done State

- WordPress ugyanazt a content baseline-t szolgálja ki
- REST endpoint valid
- Frontend WP source módban ugyanúgy néz ki, mint JSON módban
- Parity eredmény dokumentálva

### Commit javaslat
```
chore: P14.4 resync WordPress seed from JSON baseline
```

---

## P14.5 — Admin Handover Package

### Cél

Az ügyfél ténylegesen tudja szerkeszteni az oldalt.

### Jelenlegi állapot (audit alapján)

**Lokálisan léteznek:**
- `sp-benettcar/docs/benettcar-wordpress-admin-kezikonyv.doc`
- `sp-benettcar/docs/benettcar-wordpress-admin-gyors-utmutato.doc`

**Problémák:**
- Nincsenek commitolva
- `.doc` bináris formátum GitHub-on nem renderelhető, nem kereshető
- Remote-on nem igazolhatók

### Elvégzendő

**Fájlok elhelyezése:** Canonical hely: `sp-benettcar/docs/`

**Formátum döntés:**
- A `.doc` fájlok commitolhatók, de **kötelező kísérő `.md` index fájl** is létrehozandó
- Minimum: `sp-benettcar/docs/admin-guide-index.md` — listázza az elérhető dokumentumokat és tartalmaikat
- Ez biztosítja, hogy remote audit mindig lehetséges legyen

**Szükséges dokumentumok:**
- `benettcar-wordpress-admin-kezikonyv.doc` — teljes admin kézikönyv
- `benettcar-wordpress-admin-gyors-utmutato.doc` — gyors útmutató
- `admin-guide-index.md` — **ÚJ, kötelező** kísérő index
- Handover checklist (lehet a relase note része)
- Known limitations dokumentum
- Maintenance note

**Admin guide tartalmi ellenőrzés:**
- [ ] Belépés
- [ ] Homepage szerkesztés
- [ ] Hero editing
- [ ] Brand logó editing
- [ ] Gallery editing
- [ ] Services editing
- [ ] Service detail editing
- [ ] About editing
- [ ] Team editing
- [ ] Assistance editing
- [ ] Contact editing
- [ ] Map editing
- [ ] WordPress menus: `spektra-primary`, `spektra-footer`
- [ ] Kép csere folyamata
- [ ] Mentés/frissítés
- [ ] Mit ne módosítson az ügyfél
- [ ] Contact form státusz (döntés P14.6 alapján)
- [ ] Fejlesztői feladatok listája

**Screenshot rögzítési sorrend:**

| Mikor | Mit lehet rögzíteni |
|---|---|
| P14.5-ben | ACF mezők, accordion preview, menus, gallery, services — P13.5 óta stabilak |
| P14.6 után | Contact form plugin admin felülete (CF7 / WPForms) |
| P14.7 után | Production WP screenshotok (Rackforest-en futó végleges rendszer, éles URL-ek) |

A kézikönyv **szövege és screenshotjainak ~90%-a** elkészíthető P14.5-ben. A **contact szekció screenshotját** P14.6 után kell pótolni. Ha a kézikönyv a production admin URL-t és logint is tartalmazza, az éles screenshotokat P14.7 után kell véglegesíteni.

**Screenshot ellenőrzés (P14.5 scope):**
- Egyeznek-e az aktuális admin UX-szel
- Accordion preview látszik-e
- Magyar labelek frissek-e
- Contact szekció screenshot: **P14.6 után pótolni**

### Done State

- Admin guide commitolva és remote-on elérhető (contact rész P14.6 után véglegesítve)
- Quick guide commitolva és remote-on elérhető
- `admin-guide-index.md` létezik és linkeli a dokumentumokat
- Screenshotok aktuálisak (contact screenshot P14.6 utáni állapotot tükröz)
- Known limitations dokumentáltak

### Commit javaslat
```
docs: P14.5 add BenettCar admin handover package
```

---

## P14.6 — Contact Form Decision

### Cél

Ne legyen félrevezető űrlap az átadott oldalon.

### Jelenlegi állapot (audit alapján)

A `bc-contact.component.tsx` kizárólag lokális React state-et használ:
```tsx
onSubmit={(e) => {
  e.preventDefault()
  setSubmitted(true)  // ← nincs fetch, nincs backend
}}
```
Nincs backend submit, nincs email küldés, nincs API integration. A success state megtévesztő.

### Döntési opciók

**Option A — Valódi submit implementáció**
- Backend endpoint + email küldés
- GDPR/spam scope belép
- Időigényesebb
- Production szempontból korrekt

**Option B — Mailto / phone-first contact**
- Űrlap helyett CTA: telefon, email, térkép
- Nincs hamis success
- Nincs email deliverability probléma
- Nincs backend scope
- Egyszerűbb, gyorsabb, átadásbiztosabb

**Option C — Vizuális placeholder, dokumentálva**
- Csak akkor elfogadható, ha az UI NEM állítja, hogy az üzenet elküldésre került
- Ügyfélnek egyértelműen át kell adni, hogy nincs bekötve

### ✅ Döntés (2026-04-25)

**Választott opció: Option A — Valódi backend submit, WordPress pluginnal**

### ✅ Implementáció kész (2026-05-14)

> **Részletes 7-fázis implementációs terv:** [`implementation/p14-6-contact-form-formhandler.md`](../implementation/p14-6-contact-form-formhandler.md) — architektúra döntések, fázis-szintű deliverables + success criteria, párhuzamosítási lehetőségek, bukási módok + mitigáció.

A P14.7 stage-ready elérése után, a contact form implementáció **architektúra szintjén** revidálódott. A korábbi CF7-közvetlen / WPForms ötlet helyett a Spektra platform read-oldali `DataSource` absztrakciójának tükör mintáját követjük — a contact form is **driver-független** legyen, hogy ne kötődjön kizárólag WordPress-hez, miközben a BenettCar production driverét **CF7 plugin** adja (proven, kliens-szerkeszthető, beépített spam-védelem).

**Új implementációs megközelítés:**

- **Frontend: `FormHandler` driver-absztrakció** a `@spektra/data` package-ben
  - Interface: `FormHandler.submit(formId, data): Promise<FormSubmitResult>`
  - Driver-választás `VITE_FORM_HANDLER` env var-on keresztül (mint `VITE_DATA_SOURCE`)
  - React komponens `useFormHandler()` hook-ot hív, UI változatlan marad

- **Default driver: `wp-spektra`** (a mostani Rackforest WP backend use case)
  - POST `/wp-json/spektra/v1/contact` → `spektra-api` plugin új endpoint
  - PHP-szintű validáció + honeypot + `wp_mail()` (a `WP Mail SMTP` plugin SMTP-jén keresztül)
  - Response shape: a mi kontraktusunk (`{ status: 'ok' }` / `{ status: 'error', field?, message }` / `{ status: 'rate_limited' }`)
  - Előny CF7-tel szemben: nincs külön CORS namespace beállítás, response shape match-el a React state-eknek

- **Alternatív driverek** (a kliens / jövőbeli telepítések szabadsága):
  - `web3forms` — static deployment (ingyenes 3rd-party, no backend needed)
  - `formspree` — static deployment alternatíva (ingyenes 50/hó)
  - `mailto` — zero-infra fallback (mailto: link megnyitás)
  - `noop` — preview/Storybook (console.log + fake success)

- **Miért ez a megközelítés a CF7 helyett:**
  - **CMS-független architektúra:** a kliens (vagy egy következő fejlesztő) `.env` változó cserével át tudja kapcsolni a contact form-ot más backend-re — ugyanúgy, ahogy `VITE_DATA_SOURCE=json|wordpress` váltás működik
  - **Static deployment lehetséges:** ha valaha WP nélküli static export-ot akarunk (`VITE_DATA_SOURCE=json` + `VITE_FORM_HANDLER=web3forms`), a contact form akkor is működik backend nélkül
  - **CORS egyszerűbb:** ugyanaz a `spektra/v1` namespace, mint a read-oldal — már beállított CORS működik
  - **Response shape kontrollált:** field-specific error mapping a React state-be, nem kell CF7-ből visszafejteni a `mail_sent` / `validation_failed` állapotokat

További implikációk:
- Spam protection: handler-szintű (honeypot a `wp-spektra`-ban + service-szintű a `web3forms`-ban / `formspree`-ben)
- GDPR: UI-oldali (checkbox), nem handler-szintű
- Success/error state: handler `FormSubmitResult` válasza alapján
- A P14.6 scope nagyobb mint CF7-tel volt, de a strukturális nyereség (CMS-függetlenség, static deploy lehetőség) megéri

### Elvégzendő (revidált 2026-05-14)

**Architektúra (`@spektra/data`):**
1. `forms.ts` — `FormHandler` interface + `FormSubmitResult` types + `createFormHandler()` factory
2. `useFormHandler()` hook (mint a `useSiteData()`)

**Driver implementációk:**
3. `WpSpektraFormHandler` (default) — POST `/wp-json/spektra/v1/contact`
4. `Web3FormsHandler` — static deploy alternatíva
5. `MailtoHandler` — zero-infra fallback
6. `NoOpFormHandler` — preview/Storybook

**WP-Spektra backend (`sp-infra/plugin/spektra-api/`):**
7. `class-contact-controller.php` — POST endpoint, schema, validation, honeypot, rate limit, `wp_mail()`
8. Route registration a `class-rest-controller.php`-ban

**SMTP setup (Rackforest):**
9. `kapcsolat@benettcar.hu` cPanel email account létrehozás (jelszó → `deployment-runbook.md` credentials katalógusába)
10. WP Mail SMTP plugin telepítés + Rackforest SMTP konfig (`mail.benettcar.hu:465 SSL`)
11. WP Mail SMTP Email Test PASS

**Frontend integráció:**
12. `bc-contact.component.tsx` `onSubmit` → `handler.submit()` (UI nem változik)
13. GDPR checkbox required (link a `#privacy` modalra)
14. Honeypot mező (rejtett input)
15. Field-specific error mapping a React state-be

**Validáció:**
16. Valódi end-to-end submission → email megérkezés `kapcsolat@benettcar.hu`-ra
17. Spam mappa ellenőrzés
18. Rate limit teszt

**Dokumentáció:**
19. Admin guide frissítése a contact form státuszával + screenshot
20. `sp-docs/guides/form-handler-drivers.md` (új) — driver-katalógus, env-konfig, deployment-specifikus választás guide
21. `deployment-runbook §12` frissítése a `FormHandler` mintára

### Done State

- Contact form valódi submit-ot küld
- Email megérkezik a megadott címre
- Success state csak valódi API success esetén jelenik meg
- Error state kezeli a hálózati/szerver hibát
- GDPR szöveg az űrlapon
- Admin guide frissítve

### Commit javaslat
```
fix: P14.6 clarify BenettCar contact flow
```

---

## P14.7 — Staging / Production Readiness

### 🟡 Status: IN PROGRESS (2026-05-14)

**Stage-ready elérve.** Tag: [`p14.7-stage-ready`](https://github.com/Csharlie/sp-benettcar/releases/tag/p14.7-stage-ready) (sp-benettcar `b8054fb` + sp-infra `2202032`).

**Élesedett:**
- WP backend: `https://wp.benettcar.hu` (spektra-api + spektra-config + ACF Pro)
- Frontend: `https://benettcar.hu/v2/` (cookie auth gate mögött)
- REST endpoint külső hálózatról `200 OK`, mind a 10 section serve-elve
- CORS: `https://benettcar.hu` + `https://www.benettcar.hu` mindkettő preflight PASS
- Brand média WP Media Library-ben, alt text parity OK
- SSL zöld mindkét domainre (Rackforest AutoSSL)
- v2 auth gate: `login.php` + `auth-check.php` + `.htaccess` cookie-alapú, 7-nap TTL
- Mobile valódi eszköz smoke test PASS (iPhone 12, Pixel 5)
- PHP 8.4 kompatibilitás javítva sp-infra-ban (`true|\WP_Error` → `bool|\WP_Error`)
- Dokumentáció: `docs/deployment-runbook.md` (gitignored, 1925 sor) + `docs/deployment-guide.md` (gitignored, 235 sor)

**Nyitva:**
- SMTP konfiguráció + `kapcsolat@benettcar.hu` email account — P14.6 függvénye
- P14.6 contact form integráció — `FormHandler` driver-absztrakció (`@spektra/data`-ban), default `wp-spektra` driver
- Auth gate disable workflow — átadáskor (runbook §10.7)
- Decision: átadáskor a v2 marad `/v2/`-n VAGY átköltözik `/` root-ra (a régi v1-et felváltva)

### Cél

Az ügyfél publikus/staging URL-en ellenőrizhesse az oldalt.

### ✅ Infrastruktúra döntés (2026-04-25)

**Hoster:** Rackforest
**Domain:** `benettcar.hu`
**Modell:** A teljes BenettCar (frontend + WordPress) a `benettcar.hu` Rackforest-en hostolt domainre kerül.

Implikációk:
- **Deploy mechanizmus:** Rackforest tipikusan FTP/SFTP vagy cPanel file manager alapú — automatikus CI/CD pipeline nem valószínű
- **Frontend build artifact** (`dist/`) feltöltése SFTP-vel vagy cPanel-en keresztül
- **WP telepítés** szintén Rackforest-en (cPanel → Softaculous vagy manuális)
- **SSL:** Rackforest-en elérhető Let's Encrypt vagy fizetős SSL — ellenőrizni kell
- **SMTP:** Rackforest hosting levelezőszerver vagy külső SMTP (pl. SendGrid) — ez a P14.6 Option A-hoz szükséges
- **Staging vs production:** Döntendő, hogy lesz-e külön staging subdomain (pl. `staging.benettcar.hu`) vagy közvetlenül production-re megy

### CORS konfiguráció

A `sp-benettcar/infra/config.php` jelenlegi állapota:
```php
'allowed_origins' => [
    'http://localhost:5174',  // ← csak lokális
],
```

Production origineket hozzá kell adni — **ne a platformban, hanem a kliens overlay configban**.

```php
'allowed_origins' => [
    'http://localhost:5174',       // lokális dev
    'https://benettcar.hu',        // production
    'https://www.benettcar.hu',    // www alias (ha van)
    // 'https://staging.benettcar.hu',  // staging (ha lesz)
],
```

### Elvégzendő

- [x] Staging subdomain döntés: **`wp.benettcar.hu` (WP backend) + `benettcar.hu/v2/` (frontend cookie-gated)** — nincs külön staging subdomain, a v2 alkönyvtár szolgál pre-handover stage-ként
- [x] Rackforest cPanel hozzáférés és WP telepítés (`wp.benettcar.hu`, Softaculous, PHP 8.x)
- [x] Frontend build (`npm run build`) → `dist/` feltöltése FTP-vel (`/public_html/v2/`)
- [x] WP REST plugin aktiválás production-ön (`spektra-api` + `spektra-config` overlay)
- [x] CORS `allowed_origins` frissítés production URL-lel (`benettcar.hu` + `www.benettcar.hu`)
- [x] `VITE_DATA_SOURCE=wordpress` + production WP URL a deployed configban (`.env.production`)
- [x] SSL tanúsítvány ellenőrzés (Rackforest AutoSSL zöld mindkét domainre)
- [x] WP REST endpoint külső hálózatról: `https://wp.benettcar.hu/wp-json/spektra/v1/site` → `200 OK`
- [x] Media URL-ek ellenőrzése production WP-n (brand logók + képek alt text parity)
- [ ] SMTP konfiguráció (P14.6 Option A miatt kötelező) — `kapcsolat@benettcar.hu` cPanel email account + WP Mail SMTP plugin
- [ ] Admin user átadás — kliensnek külön csatornán (1Password / Signal)
- [x] Backup stratégia (Rackforest backup policy vagy WP plugin) — dokumentálva: `sp-benettcar/docs/deployment-runbook.md §15`
- [x] Update responsibility rögzítése (WP core, plugin, témák) — `deployment-runbook.md §16.4` (átadás előtt írásban a klienssel)
- [x] PHP 8.4 kompatibilitás (`class-rest-controller.php` `true|\WP_Error` → `bool|\WP_Error`) — sp-infra commit `2202032`, troubleshooting: [`php-84-true-type-literal.md`](../troubleshooting/php-84-true-type-literal.md)
- [x] v2 auth gate (cookie-alapú előzetes hozzáférés) — `login.php` + `auth-check.php` + `.htaccess`, credentials a `deployment-runbook §2`-ben
- [x] Deployment runbook + guide (`docs/deployment-runbook.md` 1925 sor + `docs/deployment-guide.md` 235 sor) — gitignored, konkrét credentials inline
- [ ] Auth gate disable workflow (átadáskor)

### Done State

- Staging URL működik
- CORS nincs hiba
- REST endpoint elérhető staging-ről
- Képek betöltődnek
- Ügyfél review-ra alkalmas

### Commit javaslat
```
chore: P14.7 staging readiness and CORS config
```

---

## P14.8 — Final Smoke Test & Release Freeze

### ✅ DONE (2026-05-14)

**Baseline commit hashek:**
- sp-platform: `3ab9768`
- sp-benettcar: `00d0d11`
- sp-infra: `2202032`
- sp-docs: `70905cd`

**Tag:** `p14.8-release-freeze` (sp-benettcar)

### Frontend checklist

- [x] `npm run build` PASS
- [x] `npm test -- --run` PASS (33/33)
- [x] Console error nincs
- [x] Mobile layout rendben
- [x] Desktop layout rendben
- [x] CTA-k működnek
- [x] Menu anchorok működnek
- [x] Képek betöltődnek
- [x] Nincs broken layout
- [x] Nincs placeholder tartalom

### Data checklist

- [x] WordPress source render rendben
- [x] REST endpoint valid (`https://wp.benettcar.hu/wp-json/spektra/v1/site` → 200, site + navigation + 1 page)

### Contact form checklist

- [x] Submit → email beérkezik `info@benettcar.hu`-ra
- [x] GDPR checkbox required
- [x] CORS preflight PASS (`Access-Control-Allow-Origin: https://benettcar.hu`)
- [x] `rate_limited` state (spam detect)
- [x] `error` state (validation hiba, field-specific)

### Release checklist

- [x] Baseline commit hash-ek rögzítve (mind a 4 repo — fent)
- [x] Staging URL: `https://benettcar.hu/v2/` (cookie auth gate)
- [x] WP admin URL: `https://wp.benettcar.hu/wp-admin`
- [ ] Credentials átadás külön csatornán — **USER ACTION**
- [x] Backup stratégia: Rackforest napi backup (`deployment-runbook §15`)
- [ ] Auth gate disable — átadáskor **USER ACTION**
- [ ] Admin kézikönyv contact form screenshot pótlása — **USER ACTION (P14.5 nyitott pont)**

### Done State

- BenettCar átadható ✅
- Nincs ismert hard blocker ✅
- Release freeze kihirdetve ✅

---

## P14.9 — Documentation Sync Across Repos

### Cél

A remote dokumentáció ne mondjon ellent a tényleges állapotnak.

### `sp-docs` teendők

- [ ] Ez a fájl (`phase-14-benettcar-handover-release.md`) folyamatosan frissítve
- [ ] `phase-12-strategy-overview.md` P12.6 státusz javítása:
  - Táblában CLOSED, de a szekcióban `[IN PROGRESS]` — **ezek ellentmondanak egymásnak, javítandó**
- [ ] P13.5 admin handover state bevezetése az overview-ba (ha hiányzik)
- [ ] JSON → WP resync workflow dokumentálása
- [ ] Contact form decision dokumentálása
- [ ] Staging/production checklist dokumentálása
- [ ] Végső commit hash-ek beillesztése

### `sp-infra` teendők

- [ ] `README.md` frissítése — jelenlegi szöveg (elavult):
  > "This repo contains NO client-specific code *except* `acf/sections.php`, which currently holds Benettcar-specific (`bc-*`) section builders..."

  Helyes állapot (P11.2 óta):
  > `acf/sections.php` generic registry és dispatch. Client overlays regisztrálják saját buildereiket.

### `spektra` (sp-platform) teendők

- [ ] `eslint.config.cjs` ellenőrzése — `packages/templates/src/**` minta nem létező könyvtárra mutat:
  - Valódi package: `packages/layouts/`
  - Ez stale import boundary szabály — silent ignorálva vagy hibás
  - Javítandó: `packages/layouts/src/**`
- [ ] README naming: `@spektra/layouts` — a README-ben már helyesen szerepel, csak az eslint config stale

### `sp-benettcar` teendők

- [ ] `docs/admin-guide-index.md` létrehozása (P14.5)
- [ ] `.gitignore` Word lock pattern (P14.0)
- [ ] `.env` source switching dokumentálása (P14.1)
- [ ] `bc-client-feedback-tracker.md` lezárása (P14.2)
- [ ] Handover checklist véglegesítése

### Done State

- Remote docs tükrözik a valós rendszert
- Nincs P13.5 / Phase 14 dokumentációs lyuk
- Nincs elavult boundary állítás
- BenettCar átadás dokumentált
- `phase-12-strategy-overview.md` P12.6 státusz konzisztens

### Commit javaslat
```
docs: P14.9 sync Spektra handover documentation
```

---

## P14.10 — Non-BenettCar Follow-up Cleanup

### Státusz: DEFERRED

Ezeket a munkákat nem szabad a BenettCar átadás elé tenni — csak utána kerülnek sorra.

### Tartalom

- Phase 12 platform hardening folytatása (P12.7 scaffold/generator)
- UI adapter architecture
- AI-generated client architecture
- sp-exotica esetleges karbantartás
- sp-platform `eslint.config.cjs` templates → layouts javítás (ha P14.9 nem fedi le)
- Seed Pipeline Source Strategy Support (P12.5 PLANNED)

---

## Recommended Execution Order

> **Megjegyzés:** A sprint bontás emberi kommunikációs eszköz — a végrehajtás és nyomkövetés a P14.x phase számok alapján történik. A sprintek nem adnak hozzá új információt a phase-ekhez, csak a végrehajtási sorrendet csoportosítják emberi olvasásra.

### Sprint A — Baseline + JSON Mode
1. Remote/local státusz ellenőrzés (P14.0)
2. `.gitignore` lock file fix (P14.0)
3. JSON source mode beállítás (P14.1)
4. Build/test baseline PASS igazolása
5. `bc-client-feedback-tracker.md` sablon létrehozása

### Sprint B — Frontend Client Feedback
1. Ügyfél feedback lista közös áttekintése és jóváhagyása
2. Tracker feltöltése az összes tétellel
3. Módosítások implementálása külön promptokban, JSON source módban
4. Minden prompt végén: build/test; tracker frissítése
5. Tracker 100%-os → Sprint C

### Sprint C — Content & Media Freeze
1. Végleges content baseline review
2. Minden brand logóhoz explicit URL döntés
3. Local asset warningok kezelése
4. Navigation/footer tartalom véglegesítése
5. `seed:export:dry` lefuttatása, warning lista dokumentálása

### Sprint D — WordPress Resync
1. `seed:export`
2. WP seed import
3. Endpoint check
4. WordPress source frontend check
5. Parity dokumentálás

### Sprint E — Admin Handover
1. Admin guide és quick guide commitolása
2. `admin-guide-index.md` létrehozása
3. Screenshot frissítés
4. Known limitations dokumentálása
5. Maintenance note

### Sprint F — Release Readiness

> **Döntések lezárva (2026-04-25):** Contact form = Option A (valódi submit), Hosting = Rackforest / benettcar.hu

1. P14.7: Rackforest cPanel hozzáférés + domain setup (előfeltétel a többihez)
2. P14.7: WP telepítés production-ön, REST plugin aktiválás
3. P14.6: Backend submit implementáció (SMTP config Rackforest-en, WP `wp_mail()` / SMTP plugin)
4. P14.6: Frontend form update (fetch + valós success/error state + GDPR szöveg)
5. P14.7: Frontend build + SFTP deploy (`dist/` feltöltése)
6. P14.7: CORS config frissítés production URL-lel
7. P14.7: End-to-end validáció: REST endpoint, képek, form, CORS
8. P14.8: Final smoke test
9. P14.8: Release freeze

### Sprint G — Cross-Repo Docs Cleanup
1. `sp-docs` Phase 14 lezárás, commit hash-ek
2. `sp-infra` README drift javítás
3. `sp-platform` eslint.config.cjs templates path javítás
4. BenettCar docs index frissítés

---

## Final Done State

A BenettCar akkor tekinthető átadásra késznek, ha:

- [ ] Ügyfél visszajelzések beépítve (tracker 100%)
- [ ] JSON source frontend build/test PASS
- [ ] WordPress source ugyanazt rendereli
- [ ] Seed pipeline lefutott (export + import)
- [ ] Media assetek rendezettek, local asset warning lista dokumentált
- [ ] Contact flow döntött és korrekt (nem fake)
- [ ] Staging működik
- [ ] CORS rendben (staging + production origin)
- [ ] Admin guide remote-on commitolva és elérhető
- [ ] `admin-guide-index.md` létezik
- [ ] Quick guide remote-on commitolva és elérhető
- [ ] Known limitations dokumentáltak
- [ ] Release checklist lezárható
- [ ] Minden repo clean
- [ ] Baseline commit hash-ek rögzítve (mind a 4 repo)
- [ ] Nincs elavult docs állítás a kritikus pontokon
