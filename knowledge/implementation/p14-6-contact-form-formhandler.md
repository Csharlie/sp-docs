---
title: "P14.6 — Contact Form: FormHandler driver-abstraction (Spektra) + CF7 production driver (BenettCar)"
status: plan-complete
type: implementation
phase_introduced: "14.6"
canonical: true
last_updated: 2026-05-14
tags: [contact-form, formhandler, cf7, spektra-data, benettcar, p14.6]
---

# P14.6 — Contact Form: FormHandler driver-abstraction + CF7 production driver

## Purpose

A BenettCar contact form **CMS-független** módon kerüljön bevezetésre úgy, hogy közben a P14 átadási határidőt is teljesíti.

A platform szinten egy generikus `FormHandler` driver-absztrakció kerül a `@spektra/data` package-be — pont, ahogy a read-oldali `DataSource` (`VITE_DATA_SOURCE=json|wordpress`) működik. A BenettCar éles production driverét a **Contact Form 7** plugin adja (proven, WP admin UI-val szerkeszthető, beépített spam-védelem) — de a `FormHandler` interfész mögött **bármikor cserélhető**.

## Scope

- **Platform-szintű reusable**: `@spektra/data` package-ben `FormHandler` interface + factory + 4 driver (`cf7`, `web3forms`, `mailto`, `noop`)
- **BenettCar-szintű production**: CF7 plugin a `wp.benettcar.hu`-n + WP Mail SMTP + CORS overlay + frontend integráció
- **Érintett repók**: `sp-platform` (forms abstraction), `sp-infra` (CF7 CORS filter ha kell), `sp-benettcar` (overlay config + frontend wire), `sp-docs` (ez a doc + guides + phase update)
- **NEM tartozik ide**: WP-Spektra natív custom REST contact endpoint (jövőbeli driver, scaffold only)

---

## 1. Architektúra elv

A Spektra platform már alkalmazza a "driver mögé absztrakt" mintát az **olvasás-oldalon**:

```ts
// @spektra/data — read-side (létező)
createDataSource({ source: env.VITE_DATA_SOURCE })
  // returns JsonDataSource | WordPressDataSource
```

Az **írás-oldal (form submission)** ezt tükrözi:

```ts
// @spektra/data — write-side (új, P14.6)
createFormHandler({ driver: env.VITE_FORM_HANDLER })
  // returns Cf7FormHandler | Web3FormsFormHandler | MailtoFormHandler | NoOpFormHandler
```

**Előnyök:**
1. **CMS-független frontend** — a React komponens nem tudja és nem érdekli, mi van a backend-en
2. **Env-driven driver-csere** — `.env`-ben 1 sor változtatás bármilyen build / deploy esetén
3. **Static deployment friendly** — `web3forms` vagy `mailto` driver mellett a SPA backend nélkül is működik
4. **Progressive enhancement** — ma CF7, holnap Strapi/Sanity/custom REST, sosem kell a React UI-t újraírni

## 2. Architektúra döntések (Fázis 0)

| Kérdés | Döntés | Indoklás |
|--------|--------|----------|
| Hol éljen a `FormHandler` interface? | `sp-platform/packages/data/src/forms/` | Platform-szintű reusable — minden kliens megkapja |
| Driver-választó env var név | `VITE_FORM_HANDLER` | Tükör konvenció a `VITE_DATA_SOURCE`-szal |
| Driverek névkonvenció | `cf7` / `wp-spektra` / `web3forms` / `formspree` / `mailto` / `noop` | Driver mit reprezentál — egyszerű lowercase slug |
| Response shape kontraktus | `{ status: 'ok' } \| { status: 'error', field?, message } \| { status: 'rate_limited' }` | Diszkriminált union — TypeScript exhaustive check, field-specific error mapping |
| BenettCar éles driver | `cf7` (Contact Form 7 plugin) | Proven, kliens-szerkeszthető, beépített spam-védelem |
| CF7 specifikus CORS | Filter a `spektra-config` plugin overlay-ban, ugyanazt az `allowed_origins` listát használja | Egy plugin marad, semmi extra CORS-plugin |
| Honeypot vagy reCAPTCHA | CF7 Honeypot addon (most), reCAPTCHA opcionális átadás után | Honeypot UX-barát, nincs Google függés |
| Dev default driver | `noop` (console.log, fake success) | Lokális dev-en nincs valódi email küldés |
| Prod default driver | `cf7` | `.env.production`-ben fixálva |
| WP-Spektra driver | Csak scaffold (interface placeholder) most | Aktivációs scope-ot a kliens igénye nyitja meg |
| Test runner | Vitest (létező) | Konzisztens a többi platform package-dzsel |

---

## 3. Fázisok

### Fázis 0 — Architektúra döntések rögzítés (~30 perc)

**Cél:** Ez a doc létrehozása + cross-reference a phase-14 P14.6 szekcióban + CHANGELOG entry.

**Deliverables:**
- `sp-docs/knowledge/implementation/p14-6-contact-form-formhandler.md` (ez a fájl)
- `sp-docs/knowledge/phases/phase-14-benettcar-handover-release.md` P14.6 szekció hivatkozás
- `sp-docs/knowledge/CHANGELOG.md` [Unreleased] entry

**Success criteria:** ez a doc létezik, linkelve van phase-14-ből, CHANGELOG frissítve.

**Repo érintés:** `sp-docs` (1 commit)
**Függőség:** nincs
**Commit msg:** `docs(P14.6): plan — FormHandler driver abstraction + CF7 production driver`

---

### Fázis 1 — Platform: `FormHandler` interface + factory (~2 óra)

**Cél:** A platform-szintű absztrakció felépítése. CMS-agnosztikus, generikus, **driverek nélkül még nem használható, de tesztelhető**.

**Fájl struktúra (`sp-platform/packages/data/src/forms/`):**

```
forms/
├── types.ts              # FormHandler interface, FormSubmitResult, FormFieldData
├── factory.ts            # createFormHandler() — env-driven driver pick
├── use-form-handler.ts   # React hook: createContext + Provider + useFormHandler
├── index.ts              # public exports
└── __tests__/
    └── factory.test.ts   # env-based driver selection unit tests
```

**Kontraktus:**

```ts
// types.ts
export interface FormHandler {
  readonly id: string  // diagnostic: 'cf7' | 'noop' | 'mailto' | ...
  submit(formId: string, fields: FormFieldData): Promise<FormSubmitResult>
}

export type FormFieldData = Record<string, string | Blob | string[]>

export type FormSubmitResult =
  | { status: 'ok'; message?: string }
  | { status: 'error'; message: string; field?: string }
  | { status: 'rate_limited'; retryAfter?: number }
```

**Hook szignatúra:**

```ts
// use-form-handler.ts
export function useFormHandler(): FormHandler { /* … */ }
export function FormHandlerProvider({ handler, children }: ...) { /* … */ }
```

**Success criteria:**
- ✅ TypeScript build green
- ✅ Vitest: `factory.test.ts` env-driven branching covered (mock `import.meta.env`)
- ✅ Export `@spektra/data` package index-ből: `useFormHandler`, `FormHandlerProvider`, `FormHandler`, `FormSubmitResult`, `createFormHandler`
- ✅ Default driver `createFormHandler()` argumentum nélkül → `NoOpFormHandler`

**Repo érintés:** `sp-platform` (1 commit)
**Függőség:** Fázis 0
**Commit msg:** `feat(data): FormHandler abstraction + factory — CMS-agnostic form submission`

---

### Fázis 2 — Platform: driver implementációk (~3 óra)

**Cél:** A 4 production-kész driver. **Mind generikusak** — BenettCar-specifikus mező-mapping nincs bennük.

**Fájl struktúra:**

```
sp-platform/packages/data/src/forms/drivers/
├── noop.ts          # console.log + fake 1s delay → 'ok' (Storybook / preview)
├── mailto.ts        # window.location = mailto:... — zero-infra fallback
├── cf7.ts           # POST /wp-json/contact-form-7/v1/contact-forms/<id>/feedback
├── web3forms.ts     # POST api.web3forms.com/submit (scaffolded, nem prod-aktivált)
└── __tests__/
    ├── cf7.test.ts        # mock fetch, CF7 response shape mapping
    ├── mailto.test.ts     # mailto: URL generation
    └── noop.test.ts       # timing, success
```

**CF7 driver belső flow:**

| Input | Konverzió | Endpoint | Response mapping |
|-------|-----------|----------|------------------|
| `FormFieldData` (objektum) | `FormData` (CF7 ezt vár) | `${apiBase}/wp-json/contact-form-7/v1/contact-forms/${formId}/feedback` | lent |

**CF7 response → `FormSubmitResult` mapping:**

| CF7 `status` | Mi a jelentés | Mapped result |
|--------------|---------------|---------------|
| `mail_sent` | Sikeres küldés | `{ status: 'ok' }` |
| `validation_failed` | Mező validáció hibás | `{ status: 'error', field: invalid_fields[0].field, message: invalid_fields[0].message }` |
| `acceptance_missing` | GDPR pipa hiányzik | `{ status: 'error', field: 'gdpr-accept', message: '...' }` |
| `spam` | Honeypot triggered | `{ status: 'rate_limited' }` |
| `mail_failed` | SMTP hiba | `{ status: 'error', message: '...' }` |
| Network error | Hálózati hiba | `{ status: 'error', message: 'Hálózati hiba' }` |

**Driver env konfig:**

```env
# CF7 driver:
VITE_FORM_HANDLER=cf7
VITE_FORM_CF7_API_BASE=https://wp.benettcar.hu       # CF7 endpoint host
VITE_FORM_CF7_FORM_ID=123                            # CF7 form ID (Fázis 3 után)

# Web3Forms driver (jövőbeli):
VITE_FORM_HANDLER=web3forms
VITE_FORM_WEB3FORMS_KEY=xxx

# Mailto fallback:
VITE_FORM_HANDLER=mailto
VITE_FORM_MAILTO_TARGET=kapcsolat@benettcar.hu

# NoOp dev:
VITE_FORM_HANDLER=noop
```

**Success criteria:**
- ✅ Mind a 4 driver tesztelve mock-okkal
- ✅ `factory.ts` mind a 4 driver-t fel tudja építeni env alapján
- ✅ CF7 driver minden CF7 response branch-et leképez (fenti táblázat)
- ✅ `web3forms.ts` scaffold — minimal POST + response mapping (nem prod-aktivált, de létezik a kód)

**Repo érintés:** `sp-platform` (1 commit, vagy driver-enként 4 ha granuláris-an akarjuk)
**Függőség:** Fázis 1
**Commit msg:** `feat(data): FormHandler drivers — cf7, web3forms, mailto, noop`

---

### Fázis 3 — BenettCar WordPress oldal: CF7 + SMTP + CORS (~1 óra)

**Cél:** A WP backend felkészítése a CF7 production driver-rel. **Párhuzamosítható Fázis 1-2-vel** — független a kódbázis változásoktól.

**Lépések (Rackforest cPanel + WP admin, server-side):**

1. **`kapcsolat@benettcar.hu` cPanel email account**
   - cPanel → Email Accounts → Create
   - Jelszó generálás → mentés `deployment-runbook.md §2` Credentials katalógusba

2. **WP Mail SMTP plugin telepítés** (`wp.benettcar.hu/wp-admin` → Plugins → Add New)
   - SMTP konfig: `mail.benettcar.hu:465 SSL`
   - From: `noreply@benettcar.hu` vagy `kapcsolat@benettcar.hu`
   - **Email Test PASS** (saját Gmail-re)

3. **Contact Form 7 plugin telepítés** (Plugins → Add New)
   - + CF7 Honeypot addon

4. **CF7 form létrehozás** (Contact → Add New)
   - Form template (mezőnevek mappelődnek a frontend `FormFieldData` kulcsaira):
     ```
     <label> Név
       [text* your-name placeholder "Az Ön neve"]
     </label>
     <label> Email
       [email* your-email placeholder "email@pelda.hu"]
     </label>
     <label> Telefon
       [tel your-phone placeholder "+36 ..."]
     </label>
     <label> Üzenet
       [textarea* your-message]
     </label>
     [acceptance gdpr-accept] Elfogadom az adatvédelmi tájékoztatót [/acceptance]
     [honeypot honeypot-field]
     [submit "Küldés"]
     ```
   - **Mail template:**
     - To: `kapcsolat@benettcar.hu`
     - Subject: `[benettcar.hu] [your-name] — Új ajánlatkérés`
     - Body: HU formatted message includes name/email/phone/message
   - **Form ID rögzítés** → `.env.production` `VITE_FORM_CF7_FORM_ID`

5. **CORS header CF7 namespace-re** (`spektra-config` plugin overlay):

   `infra/config.php`:
   ```php
   'cors_extra_namespaces' => [
       '/wp-json/contact-form-7/',
   ],
   ```

   Plus a `spektra-config.php` bootstrap-jában filter:
   ```php
   add_action('rest_api_init', function() {
       add_filter('rest_pre_serve_request', function($value) {
           // Match request URI against cors_extra_namespaces
           // Apply same Access-Control-Allow-Origin logic as spektra-api
           // ...
           return $value;
       });
   });
   ```

6. **Smoke test (kívülről):**
   ```bash
   # Preflight
   curl -i -X OPTIONS \
     https://wp.benettcar.hu/wp-json/contact-form-7/v1/contact-forms/<form-id>/feedback \
     -H "Origin: https://benettcar.hu" \
     -H "Access-Control-Request-Method: POST"
   # Várt: 200 + Access-Control-Allow-Origin: https://benettcar.hu

   # POST (test data)
   curl -i -X POST \
     https://wp.benettcar.hu/wp-json/contact-form-7/v1/contact-forms/<form-id>/feedback \
     -H "Origin: https://benettcar.hu" \
     -F "your-name=Teszt" \
     -F "your-email=test@example.com" \
     -F "your-message=Teszt üzenet" \
     -F "gdpr-accept=1"
   # Várt: { "status": "mail_sent", ... } + valódi email beérkezik
   ```

**Success criteria:**
- ✅ WP Mail SMTP Email Test PASS
- ✅ CF7 admin form mentés validation error nélkül
- ✅ CF7 endpoint preflight 200 + CORS header
- ✅ CF7 endpoint direct POST → `mail_sent` + email beérkezik
- ✅ Spam mappa: NEM landol oda

**Repo érintés:**
- `sp-benettcar/infra/config.php` (CORS CF7 namespace bővítés) — 1 commit
- `sp-benettcar/infra/spektra-config-bootstrap.php` (filter) — ugyanaz a commit (vagy a `sp-infra/plugin/spektra-api`-ban általánosabb hely)
- Server-side: WP plugin telepítés FTP-vel — nem repo
- Credentials → `docs/deployment-runbook.md` (gitignored, kézzel update)

**Függőség:** független Fázis 1-2-től (párhuzamosítható)
**Commit msg:** `feat(infra): extend CORS to CF7 namespace + production SMTP setup notes`

---

### Fázis 4 — BenettCar frontend integráció (~1.5 óra)

**Cél:** A `bc-contact` komponens átkötése a `FormHandler`-re. **UI vizuálisan nem változik.**

**Változtatások:**

1. **`src/App.tsx`** — `<FormHandlerProvider>` wrap (mint a DataSource Provider)
   ```tsx
   const formHandler = createFormHandler({ driver: import.meta.env.VITE_FORM_HANDLER })
   <FormHandlerProvider handler={formHandler}>
     {/* … */}
   </FormHandlerProvider>
   ```

2. **`src/sections/bc-contact/bc-contact.component.tsx`** —
   - `useFormHandler()` hook
   - `onSubmit` átírás: `handler.submit('contact', { 'your-name': name, ... })`
   - State diszkriminált union: `'idle' | 'loading' | 'ok' | 'error'`
   - Error mapping: `result.field` → adott input `aria-invalid` + alá hibaüzenet
   - GDPR checkbox required (modal `#privacy`-re link)
   - Honeypot rejtett input:
     ```tsx
     <input
       name="honeypot-field"
       tabIndex={-1}
       autoComplete="off"
       style={{ position: 'absolute', left: '-9999px' }}
       aria-hidden="true"
     />
     ```

3. **`.env.production`** — driver konfig:
   ```env
   VITE_FORM_HANDLER=cf7
   VITE_FORM_CF7_API_BASE=https://wp.benettcar.hu
   VITE_FORM_CF7_FORM_ID=<konkrét ID a Fázis 3-ból>
   ```

4. **`.env`** — dev defaultok:
   ```env
   VITE_FORM_HANDLER=noop
   ```

**Success criteria:**
- ✅ `npm run build` zöld (TS + Vite)
- ✅ `npm test` zöld
- ✅ Helyi dev `VITE_FORM_HANDLER=noop` → submit → console.log → 'Köszönjük' state
- ✅ Helyi dev `VITE_FORM_HANDLER=cf7` env-vel + production API base-szel: valódi email küldés
- ✅ **Vizuális diff a P14.7 stage-vel: 0** (csak a viselkedés változik, az UI nem)

**Repo érintés:** `sp-benettcar` (1 commit)
**Függőség:** Fázis 1 + 2 (driver kell) + Fázis 3 (CF7 form ID kell)
**Commit msg:** `feat(bc): wire bc-contact through FormHandler (cf7 driver) — UI unchanged, real submit`

---

### Fázis 5 — End-to-end teszt mátrix (~1 óra)

**Cél:** Valódi production smoke test, dokumentálva.

**Teszt mátrix:**

| # | Scenárió | Várt eredmény |
|---|----------|---------------|
| 1 | Valódi submit `https://benettcar.hu/v2/`-ről | Email beérkezik `kapcsolat@benettcar.hu`-ra <30s |
| 2 | Submit GDPR pipa nélkül | Frontend validáció — submit nem indul |
| 3 | Submit invalid email-lel | CF7 `validation_failed` → React error state az email mező alatt |
| 4 | Submit honeypot kitöltve (bot szimuláció) | CF7 `spam` → `rate_limited` state |
| 5 | 5× gyors submit (rate limit) | Vagy CF7 spam-detect, vagy WP user-level rate limit |
| 6 | Network offline submit | `{ status: 'error', message: 'Hálózati hiba' }` |
| 7 | WP server 500 (simulált) | Generic error state, nincs unhandled exception |
| 8 | Spam mappa ellenőrzés a fogadó oldalon | Az email NEM landol spam-be |
| 9 | Mobile valódi eszközön (iPhone + Pixel) | Form submit + UI állapotok rendben |
| 10 | Cross-browser (Chrome, Firefox, Safari) | Mind a 3-ban működik |

**Success criteria:** mind a 10 ✅ vagy explicit decision why not (külön mező a `bc-tasks.md` Backlog-ban).

**Repo érintés:** semmi (csak teszt dokumentáció — eredmény a phase-14 Commit summary-be vagy implementation-log-ba)
**Függőség:** Fázis 4

---

### Fázis 6 — Dokumentáció sync (~1 óra)

**Frissítendő fájlok:**

1. **`sp-docs/knowledge/guides/form-handler-drivers.md`** (NEW)
   - Driver-katalógus, mit mikor használj
   - Env konfig minden driverre
   - Új driver írásának receptje (extension pattern)
   - CF7 driver bonyolult részei (FormData vs JSON, CORS namespace, error mapping)

2. **`sp-docs/knowledge/phases/phase-14-benettcar-handover-release.md`**
   - P14.6 → ✅ DONE státusz
   - Phase Map ✅ ikon
   - Implementation summary (mit hova vittünk be)

3. **`sp-docs/knowledge/CHANGELOG.md`**
   - [Unreleased] Added: form-handler-drivers.md
   - [Unreleased] Changed: phase-14 P14.6 DONE summary

4. **`sp-benettcar/docs/implementation-log.md` #41 bejegyzés**
   - Cél/Miért/Hogyan/Eredmény/Döntések pattern
   - Cross-ref a `sp-platform` commitra + `sp-docs/guides` doc-ra

5. **`sp-benettcar/docs/deployment-runbook.md §12`** (gitignored)
   - Cseréld az általános CF7 leírást a konkrét végleges állapotra
   - CF7 form ID + admin URL hivatkozás
   - Driver env config production-höz
   - CF7 admin szerkesztés workflow (kliensnek átadási támpont)

6. **`sp-benettcar/docs/bc-tasks.md`** (gitignored)
   - P14.6 minden item ✅
   - P14.5 Screenshot rész kibővítendő contact form képpel

7. **Admin kézikönyv frissítés** (`benettcar-wordpress-admin-kezikonyv.doc`)
   - CF7 admin szerkesztés szekció (kliens hogyan módosíthat form mezőket)
   - Email template módosítás
   - Screenshotok pótlása

**Success criteria:** mind a hét file consistent, cross-link-elt.

**Repo érintés:** `sp-docs` (1 commit) + `sp-benettcar` (1 commit)
**Függőség:** Fázis 5 (csak akkor írunk DONE-t ha tényleg megy)
**Commit msg:**
- `docs(P14.6): form-handler-drivers guide + CF7 production wiring`
- `docs(bc): implementation-log #41 — P14.6 contact form via CF7 driver`

---

### Fázis 7 — Build, deploy, tag (~30 perc)

**Lépések:**

1. **Platform package build** (mert új modult adtunk hozzá):
   ```bash
   cd sp-platform/packages/data && npm run build
   ```
2. **Sp-benettcar build:**
   ```bash
   cd sp-clients/sp-benettcar && npm run build
   ```
3. **FTP upload:**
   - `dist/index.html` → `/public_html/v2/index.html`
   - `dist/assets/index-<új hash>.css` → `/public_html/v2/assets/`
   - `dist/assets/index-<új hash>.js` → `/public_html/v2/assets/`
   - Régi hash-elt asset-ek takarítása (`deployment-runbook §9.4`)
4. **CF7 form CORS overlay deploy:**
   - `infra/config.php` → `/wp-content/plugins/spektra-config/config.php`
   - Plugin reactivation a WP admin-ban (CORS filter regisztráció)
5. **Smoke production:** Fázis 5 tesztek mind éles URL-en újra
6. **Git tag:** `p14.6-contact-form-done` (annotált, sp-platform + sp-benettcar repón)
7. **Push:** mindkét repo + tag

**Success criteria:**
- ✅ `https://benettcar.hu/v2/` contact form submit → email beérkezik
- ✅ Console: 0 error
- ✅ Auth gate továbbra is működik (semmi regression)
- ✅ Tag pusholva sp-platform + sp-benettcar repón

**Repo érintés:** csak tag — a kód már commitolt
**Függőség:** Fázis 6
**Tag msg:** `P14.6 — Contact form via CF7 driver in FormHandler abstraction. Production smoke PASS.`

---

## 4. Összesítő idővonal

| Fázis | Mit | Idő | Repo |
|-------|-----|-----|------|
| 0 | Architektúra döntések rögzítés | 30p | sp-docs |
| 1 | `FormHandler` interface + factory + hook + tests | 2ó | sp-platform |
| 2 | 4 driver implementáció + tests | 3ó | sp-platform |
| 3 | WP-oldali CF7 + SMTP + CORS setup | 1ó | sp-benettcar (config + server) |
| 4 | Frontend `bc-contact` átkötés | 1.5ó | sp-benettcar |
| 5 | End-to-end teszt mátrix | 1ó | – |
| 6 | Dokumentáció sync | 1ó | sp-docs + sp-benettcar |
| 7 | Build, deploy, tag | 30p | sp-benettcar (tag) |
| **Total** | | **~10ó** | 3 repo érintett |

## 5. Párhuzamosítható fázisok

- **Fázis 3 (server-side WP setup)** elindítható **Fázis 1-2 mellett párhuzamosan** — ezek nem függnek egymástól. Ez akár 2-3 órát megspórolhat.
- **Fázis 5 (teszt mátrix)** részben futhat **Fázis 4 közben** — egyes lokális tesztek ideje alatt.

---

## 6. Kritikus függő pontok

| # | Mi | Mikor derül ki | Hová kerül |
|---|------|----------------|------------|
| 1 | CF7 form ID | Fázis 3 közben | `.env.production` `VITE_FORM_CF7_FORM_ID` (Fázis 4) |
| 2 | `kapcsolat@benettcar.hu` SMTP password | Fázis 3 első lépés | `deployment-runbook §2` Credentials katalógus |
| 3 | Honeypot vs reCAPTCHA választás | Fázis 0 | Honeypot — átadás után kliens kérheti reCAPTCHA-t |
| 4 | Dev vs prod driver default | Fázis 0 | Dev: `noop`, Prod: `cf7` (akár `.env.local` override-tal CF7 dev-tesztelés) |
| 5 | WP-Spektra driver scope | Fázis 0 | Scaffold (üres osztály) **NEM** kerül be most — későbbi kliens-igény nyitja |

---

## 7. Mit nem old meg ez a fázis

- **Anti-spam beyond honeypot:** ha CF7 honeypot nem elég → P14.6 utáni feladat reCAPTCHA bekötés
- **Adatkezelési nyilatkozat tartalma:** az `#privacy` modal létezik (P14.3), de a CF7 form-ról erre link kell — a modal tartalom SEM ebben a phase-ben íródik
- **Email design (HTML template):** CF7 plain text mail body-t használ — HTML-es design szépítés P14.6 utáni
- **CRM integráció:** CF7 → CRM (HubSpot, Pipedrive) — Fázis 6 utáni feature

---

## 8. Bukási módok és mitigáció

| Bukás | Tünet | Mitigáció |
|-------|-------|-----------|
| CF7 endpoint CORS 404 | Frontend "Failed to fetch" | Fázis 3 filter telepítés ellenőrzés, plugin reactivation |
| CF7 form ID inkorrekt | 404 a CF7 endpoint-on | Form ID a CF7 admin URL-jéből (`post=123`) olvasandó |
| WP Mail SMTP auth failure | Email Test fail | Cf email password helyes-e, port 465 SSL nyitva-e |
| Spam mappa landolás | Email beérkezik DE spam mappába | SPF/DKIM record a Rackforest DNS-ben — átadáskor ellenőrizendő |
| HTML form mail body nem renderelődik | Email plain text | CF7 Mail tab → "Use HTML content type" pipa |
| Frontend `result.field` nem matchel | Hibaüzenet rossz mező alatt | CF7 form field name = `FormFieldData` kulcs (1:1) |

---

## 9. Cross-references

- **Phase roadmap:** [`phases/phase-14-benettcar-handover-release.md`](../phases/phase-14-benettcar-handover-release.md) — P14.6 szekció
- **Read-side mirror absztrakció:** `@spektra/data` `createDataSource()` (P8.3 — `ba52c9f`)
- **Korábbi P14 commitok:**
  - `p14.7-stage-ready` tag (sp-benettcar `b8054fb` + sp-infra `2202032`)
  - Implementation log: [`sp-benettcar/docs/implementation-log.md`](https://github.com/Csharlie/sp-benettcar/blob/main/docs/implementation-log.md) #40
- **Deployment runbook §12 (Contact form & SMTP):** `sp-benettcar/docs/deployment-runbook.md` (gitignored, helyileg)
- **Troubleshooting:** új driverek és CF7 specifikus issue-k → új `sp-docs/knowledge/troubleshooting/` entry-k Fázis 5 közben fellelt edge case-ekre

---

## 10. Implementáció utáni doc nyitvák

Ezeknek a doksiknak frissülnie kell, amikor P14.6 ✅ DONE lesz:

- [ ] Ez a doc → `status: implementation-complete`
- [ ] [`phases/phase-14-benettcar-handover-release.md`](../phases/phase-14-benettcar-handover-release.md) → P14.6 Phase Map ✅
- [ ] [`CHANGELOG.md`](../CHANGELOG.md) → P14.6 implementation entry
- [ ] [`guides/form-handler-drivers.md`](../guides/form-handler-drivers.md) → ÚJ doc létrehozva
- [ ] `sp-benettcar/docs/implementation-log.md` → #41 entry
- [ ] `sp-benettcar/docs/deployment-runbook.md §12` → CF7 konkrét állapot
- [ ] Tag `p14.6-contact-form-done` push-olva

---

## 11. Indítás workflow

A megrendelőtől (vagy a kezdeményező fejlesztőtől) **döntés kell** az indítás módjáról:

**A)** Lineáris: Fázis 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 — egy ember, egy session
**B)** Csak platform (Fázis 1-2) most, BenettCar oldal (Fázis 3-7) külön session
**C)** Párhuzamos: Fázis 3 (server-side CF7 setup) megy előre, közben Fázis 1-2 platform munka — gyorsabb wall-clock

A B és C variánsok időt spórolnak, de több koordináció kell. Az A variánsra ajánljuk a default-ot, hacsak nincs explicit párhuzamos kapacitás.
