# Phase 11.3 — Platform + Infra Naming Refactor & Preset Scaffolding

## Purpose

P11.3 két célt szolgál:
1. A `@spektra/templates` package átnevezése `@spektra/layouts`-ra — a naming jobban tükrözi a tényleges felelősséget (layout composition, nem page template)
2. A `presets/` struktúra felépítése sp-platform és sp-infra repókban — ez a P11.4 (PHP overlay template) és a későbbi multi-client onboarding alapja

A fázis 3 repót érint: sp-platform, sp-clients/sp-benettcar, sp-infra.

---

## Érintett repók és változások

### sp-platform

| Változás | Fájl(ok) |
|---|---|
| Mappa rename | `packages/templates/` → `packages/layouts/` |
| Package name | `@spektra/templates` → `@spektra/layouts` (`package.json`) |
| Import átírás | `apps/starter/src/App.tsx`, `apps/starter/src/shell.tsx` |
| Típus rename | `TemplateShellProps` → `LayoutShellProps` (`packages/layouts/src/types.ts` definíció + re-export + 2 consumer) |
| Tailwind content glob | `apps/starter/tailwind.config.ts` — `packages/templates/` → `packages/layouts/` |
| Docs fix | `docs/cva-standard.md` — `packages/templates/*` → `packages/layouts/*` |
| Workspace | `pnpm-workspace.yaml` — `presets/*` hozzáadva, `tools/*` megmaradt |
| Preset scaffold | `presets/baseline/`, `presets/landingline/`, `presets/shopline/` — `package.json`-ok |
| Script placeholder | `scripts/scaffold.ts` — TODO placeholder |

### sp-clients/sp-benettcar

| Változás | Fájl(ok) |
|---|---|
| Import átírás | `src/App.tsx`, `src/shell/Header.tsx`, `src/shell/Footer.tsx` |
| Típus rename | `TemplateShellProps` → `LayoutShellProps` (Header.tsx, Footer.tsx) |
| Dependency | `package.json` — deps + pnpm overrides: `@spektra/templates` → `@spektra/layouts` link |
| Lockfile | `pnpm-lock.yaml` frissítve `pnpm install`-lal |

### sp-infra

| Változás | Fájl(ok) |
|---|---|
| Preset scaffold | `presets/baseline/.gitkeep`, `presets/landingline/.gitkeep`, `presets/shopline/.gitkeep` |

---

## Típus rename részletei

Egyetlen típus lett átnevezve: `TemplateShellProps` → `LayoutShellProps`.

A `LandingTemplate` komponens neve **nem változott** — az egy konkrét layout implementáció, nem a package-szintű naming kérdése.

Érintett helyek:
- `packages/layouts/src/types.ts` — definíció (interface) + `LandingTemplateProps.header`/`footer` prop típusa
- `packages/layouts/src/index.ts` — re-export
- `packages/layouts/src/LandingTemplate.tsx` — JSDoc
- `apps/starter/src/shell.tsx` — import + 2 function signature
- `sp-clients/sp-benettcar/src/shell/Header.tsx` — import + function signature
- `sp-clients/sp-benettcar/src/shell/Footer.tsx` — import + function signature

---

## Preset struktúra

```
sp-platform/presets/          sp-infra/presets/
├── baseline/                 ├── baseline/
│   └── package.json          │   └── .gitkeep
├── landingline/              ├── landingline/
│   └── package.json          │   └── .gitkeep
└── shopline/                 └── shopline/
    └── package.json              └── .gitkeep
```

sp-platform preset-ek: `@spektra/preset-baseline`, `@spektra/preset-landingline`, `@spektra/preset-shopline` — egyelőre üres `package.json`-ok, a workspace felismeri őket (`pnpm-workspace.yaml`).

sp-infra preset-ek: üres `.gitkeep` — a PHP overlay sablon tartalom P11.4-ben kerül ide.

---

## Audit findings és javítások

A rename első körben csak az sp-platform-ot és az sp-infra-t érintette. Az audit azonosította:

| ID | Szint | Finding | Fix |
|---|---|---|---|
| F1 | P1 | sp-benettcar consumer tört (`TS2307` — `@spektra/templates` not found) | Import + dep + override átírás, `pnpm install` |
| F2 | P2 | Tailwind content glob régi path-ra mutatott (csendes styling regresszió) | `tailwind.config.ts` glob frissítve |
| F3 | P2 | `pnpm-lock.yaml` stale metadata | `pnpm install --no-frozen-lockfile` mindkét repóban |
| F4 | P3 | `cva-standard.md` régi path referencia | Frissítve |

Javítás után:
- sp-platform: `pnpm build` → 8/8 task PASS
- sp-benettcar: `pnpm build` → PASS, `pnpm test` → 50/50 PASS

---

## Kizárások (szándékos)

Historikus dokumentumok nem lettek átírva — ezek az akkori állapotot tükrözik:
- `sp-platform/docs/bootstrap-platform-log.md`
- `sp-clients/sp-benettcar/docs/bc-migration-plan.md`
- `sp-clients/sp-benettcar/docs/implementation-log.md`

---

## Validáció

| Check | Eredmény |
|---|---|
| `pnpm build` (sp-platform, turbo) | 8/8 PASS |
| `pnpm build` (sp-benettcar) | PASS |
| `pnpm test` (sp-benettcar) | 50/50 PASS |
| `@spektra/templates` source-ban (excl docs/lock) | 0 találat — sp-platform + sp-benettcar |
| `TemplateShellProps` source-ban (excl docs/lock) | 0 találat — sp-platform + sp-benettcar |
| Preset package-ek workspace-ben | `pnpm install` felismeri |

---

## Megjegyzések

- A `packages/layouts/dist/` törlődött a rename során — a turbo `pnpm build` újragenerálta
- Az `apps/starter` továbbra is létezik és first-class workspace app — ez nem P11.3 scope, ha eltűnik, az külön task

---

## Következő fázisok

| Fázis | Cél | Kapcsolat |
|---|---|---|
| P11.4 | PHP overlay template — `sp-infra/presets/baseline/` feltöltése | F2 (P1) finding lezárása |
| P11.5 | Infra script paraméterezés | F3, F4 (P2) |
| P11.6 | New client onboarding guide | F2 (P1) |

---

## Done State

- [x] `packages/templates/` → `packages/layouts/` rename
- [x] `@spektra/layouts` package name
- [x] Import + típus rename sp-platform + sp-benettcar
- [x] Tailwind content glob fix
- [x] `presets/` struktúra sp-platform + sp-infra
- [x] `pnpm-workspace.yaml` `presets/*` bejegyzés
- [x] `scripts/scaffold.ts` placeholder
- [x] sp-infra `.gitkeep` preset mappák
- [x] Turbo build 8/8 PASS
- [x] sp-benettcar build + test 50/50 PASS
- [x] Audit findings (F1–F4) javítva
