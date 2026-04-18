---
title: "P13.1 — bc-services Slot-Based Admin Refactor"
status: closed
type: implementation
phase: P13.1
client: benettcar
section: bc-services
last_updated: 2026-04-19
---

# P13.1 — bc-services Slot-Based Admin Refactor

## Summary

bc-services cpt_collection pilot proved CPT collection viability, but was superseded for BenettCar handover UX by Slot-Based ACF Free Admin. The frontend SiteData contract remains unchanged.

## Motivation

A P12.6 CPT collection implementáció technikailag valid volt, de az editor UX nem volt elfogadható kliens handover-hez:
- Section title/subtitle a Homepage-on, service itemek külön CPT menüben
- Két admin location → split editing, kliens számára nem egyértelmű

## Source Strategy Correction

| Content type | Strategy |
|-------------|----------|
| Bounded landing content | slot-based fixed collection |
| Unbounded managed content | cpt_collection |
| Optional Pro convenience | acf_repeater_optional |
| Future premium UX | Gutenberg Section Editor |

## Implementation

### Admin Model (before → after)

**Before (P12.6):**
- Services CPT menü (show_ui: true, show_in_menu: true)
- Homepage: title + subtitle + legacy repeater (ne szerkeszd)
- Service items: külön CPT editor

**After (P13.1):**
- Homepage BC Services: title + subtitle + 6 slot mező (title/icon/description)
- Services CPT: hidden (show_ui: false, show_in_menu: false), retained for fallback
- Legacy repeater: removed from visible admin, data retained in DB

### Slot Fields

6 slot, mindegyik 3 mezővel:
- `bc_services_service_{n}_title` (text)
- `bc_services_service_{n}_icon` (text)
- `bc_services_service_{n}_description` (textarea, 3 rows)

ACF `message` típusú elválasztó mezők slotcsoportonként az admin olvashatóság érdekében.

### Builder Source Priority

1. Slot-based fields (Homepage) — `spektra_bc_get_service_slots()`
2. Hidden CPT fallback — `spektra_bc_get_services()`
3. Legacy repeater fallback — `bc_services_services`
4. null — no valid services

### Files Changed (sp-benettcar)

| File | Change |
|------|--------|
| `infra/acf/sections/bc-services.php` | Repeater → 6 slot fields + message separators + instructions |
| `infra/acf/cpt-service.php` | show_ui/show_in_menu → false |
| `infra/acf/builders.php` | Added `spektra_bc_get_service_slots()`, updated builder priority |
| `infra/migrations/migrate-services-to-slots.php` | New: CPT/repeater → slot field migration |

### Validation

| Case | Scenario | Result |
|------|----------|--------|
| A | Slots populated | PASS — builder uses slots, parity exact |
| B | Slots empty, CPT exists | PASS — falls back to CPT |
| C | Slots empty, CPT draft, repeater exists | PASS — falls back to repeater |
| D | No sources | null — section skipped (existing behavior) |

## Future Direction

- Slots: short-term handover-friendly baseline
- Gutenberg Section Editor: future direction for premium admin UX
- SiteData: stable contract enabling future source replacement without frontend changes
- Seed mapping update OUT OF SCOPE for P13.1

## Rollback

`git revert` on sp-benettcar P13.1 commit returns to CPT-first model. CPT posts retained, repeater data retained. SiteData contract unchanged.
