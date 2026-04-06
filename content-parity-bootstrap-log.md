# Phase 8.5 — Content Parity Bootstrap — Log

Kronológikus napló (fordított sorrend — legújabb felül): mi jött létre, mikor, miért.

> Workstream definíció: [content-parity-bootstrap.md](content-parity-bootstrap.md)
> TODO checklist: [content-parity-bootstrap-todo.md](content-parity-bootstrap-todo.md)

---

## Jelenlegi állapot

> Utolsó frissítés: P8.5.1 Architecture Closure (#2)

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
| `seed/import-seed.sh` | sp-infra | WP-CLI = infra művelet |
| `seed/verify-parity.ts` | sp-infra | parity check = platform-szintű gate |

### Commit history

| # | Hash | Leírás | Mini-phase |
|---|---|---|---|
| 1 | `41b6645` | docs(P8.5.1): architecture closure — workstream def | P8.5.1 |
| 2 | `b64157a` | docs(P8.5.1): add content parity bootstrap TODO checklist | P8.5.1 |

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
