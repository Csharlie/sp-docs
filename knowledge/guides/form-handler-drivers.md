---
title: FormHandler Drivers — katalógus, env-konfig, extension recept
type: guide
status: current
created: 2026-05-14
tags: [formhandler, cf7, spektra-data, contact-form, p14.6]
---

# FormHandler Drivers

A Spektra platform form-submission rétege a read-oldali `DataSource` (`VITE_DATA_SOURCE`) tükörképe: egy generikus `FormHandler` interfész mögé kerülnek a konkrét backend-ek. A driver-t `VITE_FORM_HANDLER` env var választja ki — build-time döntés, futásidőn nem változik.

**Architektúra-hivatkozás:** [`implementation/p14-6-contact-form-formhandler.md`](../implementation/p14-6-contact-form-formhandler.md)

---

## Interface

```ts
// @spektra/data
interface FormHandler {
  readonly id: string
  submit(formId: string, fields: FormFieldData): Promise<FormSubmitResult>
}

type FormFieldData = Record<string, string | Blob | string[]>

type FormSubmitResult =
  | { status: 'ok'; message?: string }
  | { status: 'error'; message: string; field?: string }
  | { status: 'rate_limited'; retryAfter?: number }
```

A `result.field` opcionális: ha jelen van, az adott nevű input alá kerül a hibaüzenet a UI-ban.

---

## Elérhető driverek

### `noop` — fejlesztői alapértelmezett

```env
VITE_FORM_HANDLER=noop
```

- `console.log`-olja a beküldött adatokat
- 1 másodperc szimulált késleltetés után `{ status: 'ok' }` választ ad
- Nem küld valódi emailt, nem igényel backend-et
- **Mikor használd:** lokális fejlesztés, preview, Storybook

---

### `cf7` — Contact Form 7 (production driver, BenettCar)

```env
VITE_FORM_HANDLER=cf7
VITE_FORM_CF7_API_BASE=https://wp.benettcar.hu
VITE_FORM_CF7_FORM_ID=24
```

POST-ol a CF7 REST endpoint-ra:
```
{apiBase}/wp-json/contact-form-7/v1/contact-forms/{formId}/feedback
```

Kérés törzse: `multipart/form-data` (CF7 ezt várja).

**CF7 response → FormSubmitResult mapping:**

| CF7 `status` | FormSubmitResult |
|---|---|
| `mail_sent` | `{ status: 'ok', message }` |
| `validation_failed` | `{ status: 'error', field: invalid_fields[0].field, message }` |
| `acceptance_missing` | `{ status: 'error', field: 'gdpr-accept', message }` — **NEM `rate_limited`** |
| `spam` | `{ status: 'rate_limited' }` |
| `mail_failed` | `{ status: 'error', message }` |
| hálózati hiba / JSON parse error | `{ status: 'error', message: 'Hálózati hiba…' }` |

**Headless quirk — `_wpcf7_unit_tag`:**
CF7 REST endpoint megköveteli a `_wpcf7_unit_tag` anti-CSRF mezőt. Normál WP-page beágyazásnál a shortcode rendereli. Headless integrációban a driver automatikusan injektálja:
```
_wpcf7_unit_tag = wpcf7-f{formId}-p1-o1
_wpcf7 = {formId}
```
A hívó felülbírálhatja a `fields` objektumban ha más `containerPostId`-t kell küldeni.

**CF7 Honeypot addon — headless inkompatibilis:**
A CF7 Honeypot addon (`contact-form-7-honeypot`) minden form-betöltéskor egy titkosított nonce-t generál egy rejtett mezőben. Headless integráció nem tölti be a CF7 form HTML-jét → nincs nonce → az addon minden beküldést `spam`-nek jelöl. **Megoldás: ne legyen aktív a CF7 Honeypot addon.** A React komponensben lévő off-screen honeypot mező kliens-oldali védelmet biztosít.

**Kötelező CORS:** a CF7 namespace (`/contact-form-7/`) alapból nincs benne a Spektra CORS szűrőben. A `spektra-config-bootstrap.php` `rest_pre_serve_request` filter-je terjeszti ki — `config.php`-ban:
```php
'cors_extra_namespaces' => ['/contact-form-7/'],
```

---

### `mailto` — zero-infra fallback

```env
VITE_FORM_HANDLER=mailto
VITE_FORM_MAILTO_TARGET=kapcsolat@benettcar.hu
```

`window.location` = `mailto:` URL-t állít be, a form adatait query paraméterként fűzi hozzá. A böngésző megnyitja az emailklienst.

- Nem igényel semmilyen backendet
- Megbízhatóság: a felhasználó emailkliensétől függ (nem kézbesített email nincs)
- **Mikor használd:** emergency fallback ha a CF7 backend nem elérhető, vagy WP-mentes static deploy-ban

---

## Env-konfig összefoglaló

| Driver | `VITE_FORM_HANDLER` | Kötelező env-ek |
|---|---|---|
| noop | `noop` | — |
| cf7 | `cf7` | `VITE_FORM_CF7_API_BASE`, `VITE_FORM_CF7_FORM_ID` |
| mailto | `mailto` | `VITE_FORM_MAILTO_TARGET` |

`.env` (dev default):
```env
VITE_FORM_HANDLER=noop
```

`.env.production` (BenettCar):
```env
VITE_FORM_HANDLER=cf7
VITE_FORM_CF7_API_BASE=https://wp.benettcar.hu
VITE_FORM_CF7_FORM_ID=24
```

---

## Új driver írása

1. Hozz létre egy fájlt: `sp-platform/packages/data/src/forms/drivers/my-driver.ts`

```ts
import type { FormHandler, FormFieldData, FormSubmitResult } from '../types'

export interface MyDriverConfig {
  apiKey: string
}

export function createMyDriver(config: MyDriverConfig): FormHandler {
  return {
    id: 'my-driver',
    async submit(formId, fields): Promise<FormSubmitResult> {
      // ... valódi fetch / logika
      return { status: 'ok' }
    },
  }
}
```

2. Add hozzá a `factory.ts` switch ágához:
```ts
case 'my-driver':
  return createMyDriver({ apiKey: ... })
```

3. Exportáld az `index.ts`-ből.

4. Írj vitest unit tesztet `__tests__/my-driver.test.ts`-ben.

**Nem szabad:** React-ot importálni a driver-ből. A `@spektra/data` package pure async — React Context / hook csak a `@spektra/runtime`-ban él.

---

## React integráció

```tsx
// @spektra/runtime
import { FormHandlerProvider, useFormHandler } from '@spektra/runtime'

// App.tsx — provider wrap
<FormHandlerProvider handler={createBcFormHandler()}>
  <LandingTemplate ... />
</FormHandlerProvider>

// bc-contact.component.tsx — hook
const handler = useFormHandler()
const result = await handler.submit('contact', fields)
```

---

## Kapcsolódó docs

- [`implementation/p14-6-contact-form-formhandler.md`](../implementation/p14-6-contact-form-formhandler.md) — architektúra döntések, fázis-szintű deliverables
- [`phases/phase-14-benettcar-handover-release.md`](../phases/phase-14-benettcar-handover-release.md) — P14.6 státusz
