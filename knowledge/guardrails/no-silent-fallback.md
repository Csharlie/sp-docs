---
title: No Silent Fallback – WP Adapter hibakezelési szabály
status: draft
type: guardrail
phase_introduced: "9.1"
canonical: true
repos: [sp-platform, sp-clients]
tags: [guardrail, error-handling, wordpress-adapter, fallback, runtime, null-state]
last_updated: 2026-04-11
---

# No Silent Fallback – WP Adapter hibakezelési szabály

## Summary (Simple)

Alapszabály: ha a WordPress API hibás választ ad (non-2xx), a WP adapter **soha nem térhet vissza statikus JSON adatokkal**. Mindig `throw`-olni kell, a runtime `null` data state-et állít be, és a kliens template saját branded error UI-t jelenít meg. A csendes fallback elfogadhatatlan, mert elrejtené a valódi hibát és elavult tartalmat mutatna.

## Technical Explanation

A hibakezelési lánc három rétegen keresztül fut:

```
wordpress.ts (throw)
    ↓
context.tsx (catch → data: null, error set)
    ↓
LandingTemplate (error slot render)
    ↓
App.tsx error prop (branded UI)
```

### 1. `wordpress.ts` — throw a non-2xx válaszra

**Fájl**: `sp-platform/packages/data/src/wordpress.ts`

```ts
const response = await fetch(url);
if (!response.ok) {
  throw new Error(`WordPress API error: ${response.status} ${response.statusText}`);
}
```

- `response.ok` false → azonnal `throw`
- Nincs catch blokk, ami JSON fallback-ot aktiválna
- A hiba felfelé propagál a runtime context-be

### 2. `context.tsx` — null state, nem fallback

**Fájl**: `sp-platform/packages/runtime/src/context.tsx`

```ts
try {
  const data = await adapter.fetch();
  setState({ data, loading: false, error: null });
} catch (error) {
  setState({ data: null, loading: false, error });
}
```

- `data: null` — NEM `data: fallbackJson`
- Az `error` objektum megtartott, a template megkapja
- Nincs `VITE_DATA_SOURCE` runtime-check, nincs adapter-csere hibára reagálva

### 3. `LandingTemplate.tsx` — error slot

**Fájl**: `sp-platform/packages/templates/src/LandingTemplate.tsx`

A template `renderError` (vagy `error`) propot fogad. Ha a prop nincs megadva, a template belső default:

```tsx
<div role="alert">{error.message}</div>
```

Ez branding nélküli fallback — platform-szinten szándékos: a kliens felel a branded megjelenésért.

### 4. Adapter factory — env csak egyszer olvasva

**Fájl**: `sp-clients/sp-benettcar/src/data/create-adapter.ts`

```ts
const source = import.meta.env.VITE_DATA_SOURCE;
// ...
export const adapter = createAdapter(source);
```

- Az env modul inicializáláskor olvasódik — nem requestenként
- Ismeretlen `source` értékre: JSON adapter + `console.warn`
- Nincs runtime adapter swap — ha egyszer WP adapter lett létrehozva, az marad

### Teljes hibakezelési lánc — diagram

```
[HTTP kérés]
    │
    ▼
wordpress.ts
    │ non-2xx?
    ▼
throw new Error('WordPress API error: 503 ...')
    │
    ▼
context.tsx catch()
    │
    ├─ data: null        ← NEM fallback JSON
    ├─ loading: false
    └─ error: Error obj
         │
         ▼
LandingTemplate
    │ error prop megadva?
    ├─ igen → branded error UI (kliens felelős)
    └─ nem  → <div role="alert">{error.message}</div> (platform default)
```

## Why It Exists

A csendes fallback veszélyes mert:
1. A felhasználó nem tudja, hogy az oldal elavult statikus adatot mutat
2. CMS szerkesztők nem kapnak visszajelzést, hogy a változtatásuk nem jelent meg
3. A monitoring nem jelzi a hibát, mert a frontend "sikeresen" töltődött be
4. Újraindítás után a szokatlan állapot rejtve marad

A guardrail biztosítja, hogy **minden WP hiba látható** — vagy a felhasználónak (branded error UI), vagy a fejlesztőnek (console error + network tab).

## Implementation Notes

- **wp adapter throw**: `sp-platform/packages/data/src/wordpress.ts`
  - `if (!response.ok) throw new Error(...)`
- **runtime null state**: `sp-platform/packages/runtime/src/context.tsx`
  - catch → `{ data: null, loading: false, error }`
- **template error slot**: `sp-platform/packages/templates/src/LandingTemplate.tsx`
  - `error` prop (render function: `(err: Error) => ReactNode`)
- **factory env-at-load**: `sp-clients/sp-benettcar/src/data/create-adapter.ts`
  - env olvasás modul szinten, nem requestenként
- **branded error UI**: `sp-clients/sp-benettcar/src/App.tsx`
  - `error` prop implementálva (lásd: [p9.1-error-ui-pattern.md](p9.1-error-ui-pattern.md))

## Boundaries and Guardrails

- **Owner**: platform réteg (`sp-platform/packages/data`, `sp-platform/packages/runtime`)
- **NEM szabad**: a `wordpress.ts`-ben hibás választra fallback JSON visszaadása
- **NEM szabad**: a `context.tsx`-ben hiba esetén `data`-t nem-null értékre állítani
- **NEM szabad**: a kliens adapter factory-ban runtime adapter swap logikát bevezetni
- **NEM szabad**: az error state-et silent log-olni és az UI-t "üresnek" hagyni — explicit error UI kötelező
- A platform biztosítja az error slot infrastruktúráját; a branded megjelenés kliens felelőssége

## Related Concepts

- [p9.1-runtime-cutover-baseline.md](p9.1-runtime-cutover-baseline.md) — CP4 kontextus
- [p9.1-error-ui-pattern.md](p9.1-error-ui-pattern.md) — branded error UI implementáció
- [../phase-8.5/p8.5-parity-gate.md](../phase-8.5/p8.5-parity-gate.md) — az infra előfeltétel gate

## Open Questions / TODO

- Hálózati timeout kezelés (`AbortController`): jelenleg nincs timeout a `fetch`-en — explicit TODO a WP adapter kiegészítésére P10+.
- Retry logika: nincs bekonfigurálva — szándékos vagy hiány? Tisztázandó P10 előtt.
