# Phase 12.3 — First Runtime Migration Planning / Pilot

## Purpose

Az első runtime migration target kiválasztása az ACF Repeater dependency kiváltására a free WordPress baseline-ban. Ez planning fázis — runtime implementáció csak az elfogadott target döntés után indulhat.

## Context

P12.2 bevezette a Repeatable Content Source Strategy-t (`concepts/repeatable-content-source-strategy.md`):
- `cpt_collection` — default free baseline (CPT; taxonomy csak ahol kategorizáció indokolt)
- `fixed_slots` — bounded kis tartalom (ACF Free-kompatibilis)
- `acf_repeater_optional` — opcionális jövőbeli Pro source

13 repeater field / 11 szekció classifikálva. Stratégiai döntés: Option C (Split Tracks) — lásd Decision szekció.

---

## Decision

Option C selected: Split Tracks.

- **P12.3a**: fixed_slots technical pilot planning
- **P12.3b**: BenettCar cpt_collection design

**Rationale:**
- fixed_slots pilot alacsony kockázatú validációt ad az ACF Repeater kiváltásra
- BenettCar cpt_collection design védi a handover prioritást
- split track nem blokkolja a platform validációt, miközben a valós kliens editorial pain-t is címzi

**Fontos:** Ez a lépés a stratégiai döntést rögzíti. Egyik track sem tartalmaz runtime implementációt. P12.3a a planning lezárásakor implementációs promptot produkál. P12.3b design dokumentum marad az explicit platform-owner jóváhagyásig.

---

## Decision Context (resolved)

Az alábbi szekciók a döntési folyamat dokumentációja. Option C kiválasztásra került — a candidate-ok és kritériumok a döntés indoklását szolgálják.

### Candidate A — fixed_slots Technical Pilot (→ P12.3a)

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

### Candidate B — BenettCar cpt_collection Planning (→ P12.3b)

**Lehetséges target-ek:**
- sp-benettcar: `bc-gallery.images`
- sp-benettcar: `bc-brand.brands`
- sp-benettcar: `bc-team.members`

**Fontos: Candidate B design és migration planning dokumentum. NEM jelent CPT runtime kód implementációt — a végleges target kiválasztás explicit platform-owner jóváhagyást igényel.**

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

### Option C — Split Tracks (selected)

Option C kiválasztva. A split tracks megközelítés:
- **P12.3a**: `fixed_slots` pilot planning — ez záródik előbb.
- **P12.3b**: BenettCar `cpt_collection` design párhuzamosan indulhat, de design dokumentum marad az explicit jóváhagyásig.
- P12.3b NEM blokkolhatja P12.3a lezárását.
- Runtime implementáció továbbra is külön elfogadott implementációs promptot igényel.

### Decision Criteria (applied)

Kandidátusok összehasonlítása az alábbi dimenziók mentén történt:
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

**Eredmény:** Option C kiválasztva — mindkét dimenzió (platform validáció + handover) párhuzamosan haladhat.

---

## P12.3a — fixed_slots Technical Pilot Planning

**Candidates:**
- sp-exotica: `eb-contact.opening_hours`
- sp-exotica: `eb-about.values`

**Purpose:**
- fixed_slots source strategy bizonyítása
- egy egyszerű ACF Repeater dependency kiváltásának előkészítése ACF Free-kompatibilis bounded mezőkkel
- SiteData output változatlan marad

**Fontos:** Ez kizárólag planning. P12.3a akkor záródik, amikor az implementációs prompt elkészült. A tényleges runtime implementáció külön explicit implementációs promptot igényel.

**Required planning output implementáció előtt:**
- kiválasztott fixed_slots target
- field representation döntés: numbered fields vs textarea split vs egyéb bounded reprezentáció
- builder mapping plan
- seed impact
- SiteData parity validation plan
- rollback plan
- implementációs prompt készültségi értékelés

---

## P12.3b — BenettCar cpt_collection Design

**Candidates:**
- sp-benettcar: `bc-gallery.images`
- sp-benettcar: `bc-brand.brands`
- sp-benettcar: `bc-team.members`

**Purpose:**
- az első cpt_collection migration pattern megtervezése
- BenettCar handover stabilitás támogatása
- editorial UX javítása magas interakciójú tartalomhoz

**Fontos:** Ez kizárólag design. CPT runtime kód NEM implementálható P12.3b-ben. P12.3b ajánlást produkál, nem végleges runtime implementációs döntést. A végleges cpt_collection target kiválasztás explicit platform-owner jóváhagyást igényel implementáció előtt.

**Required planning output:**
- ajánlott első BenettCar cpt_collection target
- ajánlás indoklása
- CPT naming convention javaslat
- collection loading pattern javaslat
- builder output shape
- seed strategy impact
- SiteData parity validation plan
- migration/rollback plan
- handover impact assessment
- implementációs prompt készültségi értékelés

---

## Guardrails

- WordPress/CPT/ACF logika NEM kerülhet az sp-platform core-ba
- SiteData shape változatlan marad
- Frontend section props változatlan marad
- ACF Pro nem baseline requirement
- Meglévő kliensek megtarthatják a repeater-eket a migration-ig
- P12.3a planning-only, amíg külön implementációs prompt nem elfogadott
- P12.3b design-only, amíg később expliciten nem jóváhagyott
- Runtime kód csak elfogadott implementációs prompt után indulhat

---

## Done State

P12.3 planning lezárási feltétel:
1. Option C rögzítve mint kiválasztott stratégia
2. P12.3a target ajánlás dokumentálva és implementációs prompt biztonságosan megírható
3. P12.3b cpt_collection design ajánlás dokumentálva
4. SiteData stability validation plan mindkét track-re létezik
5. Rollback plan mindkét track-re létezik
6. Runtime kód NEM módosult ebben a planning lépésben
