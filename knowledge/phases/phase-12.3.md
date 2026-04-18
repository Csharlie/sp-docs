# Phase 12.3 — First Runtime Migration Planning / Pilot

## Purpose

Az első runtime migration target kiválasztása az ACF Repeater dependency kiváltására a free WordPress baseline-ban. Ez planning fázis — runtime implementáció csak az elfogadott target döntés után indulhat.

## Context

P12.2 bevezette a Repeatable Content Source Strategy-t (`concepts/repeatable-content-source-strategy.md`):
- `cpt_collection` — default free baseline (CPT; taxonomy csak ahol kategorizáció indokolt)
- `fixed_slots` — bounded kis tartalom (ACF Free-kompatibilis)
- `acf_repeater_optional` — opcionális jövőbeli Pro source

13 repeater field / 11 szekció classifikálva. P12.3 feladata az első migration target kiválasztása kód módosítás előtt.

---

## Candidate A — fixed_slots Technical Pilot

**Lehetséges target-ek:**
- sp-exotica: `eb-contact.opening_hours`
- sp-exotica: `eb-about.values`

**Indoklás:**
- Kis, bounded tartalom — max 5-7 item
- Nincs image handling
- Nincs taxonomy szükséglet
- Lowest-risk validáció az ACF Repeater kiváltásra
- Alkalmas a `fixed_slots` pattern bizonyítására

**Kockázat:**
- Alacsonyabb üzleti impact mint a BenettCar handover kérdések
- Nem validálja a `cpt_collection` pattern-t
- Exotica-specifikus — nem ad közvetlenül BenettCar handover értéket

---

## Candidate B — BenettCar cpt_collection Planning

**Lehetséges target-ek:**
- sp-benettcar: `bc-gallery.images`
- sp-benettcar: `bc-brand.brands`
- sp-benettcar: `bc-team.members`

**Fontos: Candidate B a P12.3 keretében design és migration planning dokumentumot jelent. NEM jelent CPT runtime kód implementációt, hacsak egy későbbi explicit implementációs prompt azt nem engedélyezi.**

**Indoklás:**
- Közvetlen BenettCar handover stabilitás támogatás
- Valós editorial UX pain megoldása
- A fontosabb `cpt_collection` pattern validálása

**Kockázat:**
- Nagyobb scope
- CPT naming convention döntés szükséges
- Collection loading pattern definiálása szükséges
- Seed strategy planning szükséges
- Erősebb parity validation szükséges

---

## Option C — Split Tracks

Ha split tracks mellett döntünk:
- **P12.3a**: `fixed_slots` pilot döntés és implementáció planning — ez záródik előbb.
- **P12.3b**: BenettCar `cpt_collection` design párhuzamosan indulhat, de design dokumentum marad az explicit jóváhagyásig.
- P12.3b NEM blokkolhatja P12.3a lezárását.
- Runtime implementáció továbbra is külön elfogadott target döntést igényel.

---

## Decision Criteria

Kandidátusok összehasonlítása:
- technikai kockázat
- üzleti / handover impact
- implementációs scope
- SiteData stabilitási kockázat
- seed pipeline impact
- editorial UX javulás
- jövőbeli scaffold/generator érték

**Döntési felelősség:**
A végső target döntés nem automatikus, ha tradeoff-ok fennállnak. A tradeoff-ot világosan kell bemutatni és egy opciót ajánlani. A végleges kiválasztás platform-owner döntés, kivéve ha az evidencia egyértelmű.

**Súlyozási irányelvek:**
- Delivery/handover fókusz: editorial UX improvement és business impact súlyozottabb
- Platform validáció fókusz: technikai kockázat és bounded scope súlyozottabb
- Hosszú távú platform érettség: scaffold/generator érték és seed impact súlyozottabb

---

## Required Decision Before Implementation

P12.3-nak az alábbiak közül kell választania:
1. `fixed_slots` pilot indítása
2. BenettCar `cpt_collection` planning indítása
3. Split tracks:
   - P12.3a `fixed_slots` pilot
   - P12.3b BenettCar `cpt_collection` design

---

## Required Output

Runtime kód módosítás előtt P12.3-nak produkálnia kell:
- kiválasztott migration target vagy planning track
- döntési indoklás
- source strategy
- elvárt WP field/source pattern
- elvárt builder pattern
- seed impact
- SiteData parity validation plan
- rollback plan
- done state
- implementációs prompt készültségi értékelés

---

## Guardrails

- WordPress/CPT/ACF logika NEM kerülhet az sp-platform core-ba
- SiteData shape változatlan marad
- Frontend section props változatlan marad
- ACF Pro nem baseline requirement
- Meglévő kliensek megtarthatják a repeater-eket a migration-ig
- Candidate B planning/design marad, hacsak explicit runtime implementáció nem engedélyezett
- Runtime kód csak az elfogadott target döntés után indulhat

---

## Done State

P12.3 planning lezárási feltétel:
1. Első migration target vagy split-track stratégia explicit kiválasztva
2. Source strategy megerősítve
3. Implementációs scope bounded
4. Validation plan definiálva
5. Rollback plan definiálva
6. Runtime implementációs prompt biztonságosan megírható
