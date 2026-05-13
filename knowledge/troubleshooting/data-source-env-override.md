---
title: VITE_DATA_SOURCE — .env.local felülírja a .env-et
status: stable
type: troubleshooting
phase_introduced: "14.4"
scope: cross-repo
repos: [sp-clients]
tags: [vite, env, env-local, adapter, wordpress, json, data-source, troubleshooting]
last_updated: 2026-05-13
canonical: true
---

# VITE_DATA_SOURCE — `.env.local` felülírja a `.env`-et

## TL;DR

**Ha azt mondod a kliensnek `.env`-ben hogy `VITE_DATA_SOURCE=wordpress`, de a böngészőben mégis a `site.ts` mock adat látszik, és a Network tabon NINCS request `benettcar.local/wp-json/spektra/v1/site`-ra — ellenőrizd az `.env.local`-t.**

Vite a `.env.local`-t magasabb prioritással tölti be mint a `.env`-et. Ha az `.env.local` `VITE_DATA_SOURCE=json`-t tartalmaz, a `.env`-ben lévő `VITE_DATA_SOURCE=wordpress` **figyelmen kívül marad**.

## A jelenség

| Tünet | Megnyilvánulás |
|---|---|
| Frontend nem frissül a WP módosítás után | `localhost:5174` változatlan, `Ctrl+Shift+R` sem segít |
| Network tab üres a REST endpoint felé | `benettcar.local/wp-json/spektra/v1/site` request nincs |
| `site.ts` mock adat látszik | A site.ts tartalom jelenik meg, nem a WP-é |
| `.env` látszólag helyes | `VITE_DATA_SOURCE=wordpress` benne van, mégsem érvényesül |
| WP REST endpoint helyes adatot ad vissza | `curl http://benettcar.local/wp-json/spektra/v1/site` az új tartalmat mutatja |
| CORS rendben | `Access-Control-Allow-Origin: http://localhost:5174` jelen van |

## Root cause

Vite env loading prioritás (magasabb → alacsonyabb):

1. `.env.local` ← **NEM commitolt, lokális override**
2. `.env.[mode].local` (pl. `.env.development.local`)
3. `.env.[mode]` (pl. `.env.development`)
4. `.env`

Ha `.env.local` definiál egy változót, az **mindenhol felülírja** az `.env`-ben lévőt — függetlenül attól hogy mit lát az ember a `.env`-ben. Vite a `.env.local`-t azért használja, mert ez a "developer's own settings, never committed" mintát követi.

## A konkrét eset (Benett Car, P14.4 átállás)

P14.1 alatt a kliens JSON adapterre lett állítva fejlesztés idejére:

```dotenv
# .env.local (P14.1 örökség)
VITE_DATA_SOURCE=json
```

P14.4-ben átállítottuk a `.env`-et WP-re:

```dotenv
# .env
VITE_DATA_SOURCE=wordpress
VITE_WP_API_BASE=http://benettcar.local
```

**De az `.env.local` ottmaradt és felülírta** → a Vite dev server `VITE_DATA_SOURCE=json`-t látta, a `createAdapter()` JSON adaptert hozott létre, a fetch sosem indult el a REST endpoint felé.

## Megoldás

1. **Listázd a env fájlokat** — ne csak a `.env`-et nézd:
   ```powershell
   ls .env*
   ```

2. Ha van `.env.local`, **olvasd is ki**:
   ```powershell
   cat .env.local
   ```

3. Ha tartalmaz `VITE_DATA_SOURCE`-ot, állítsd át vagy töröld:
   ```dotenv
   # .env.local
   VITE_DATA_SOURCE=wordpress
   ```

4. **Indítsd újra a Vite dev servert** — Vite nem mindig veszi fel automatikusan a `.env*` változtatást:
   ```powershell
   # Ctrl+C, majd:
   npm run dev
   ```

5. **Hard refresh a böngészőben** (`Ctrl+Shift+R`).

6. **Verifikáció:** Network tab → keresd a `benettcar.local/wp-json/spektra/v1/site` requestet. 200-as válasz = WP adapter aktív.

## Prevention — agent-eknek

Mielőtt elkezdesz adapter-állapotot debugolni (JSON ↔ WordPress váltás, REST endpoint kapcsolat, adat-frissülés), **mindig** futtasd:

```bash
ls -la <client-dir>/.env*
cat <client-dir>/.env
cat <client-dir>/.env.local 2>/dev/null || echo "no .env.local"
```

Vite env loading prioritás a hivatalos doksiban: <https://vitejs.dev/guide/env-and-mode.html#env-files>.

## Kapcsolódó

- [[seed-pipeline-blockers]] — seed pipeline boldogtalanságok
- `sp-clients/sp-benettcar/infra/env/README.md` — env változók listája és prioritás
- `src/data/create-adapter.ts` — az adapter switch logika
