---
adr: ADR-005
title: "Capture Vide - Stockée"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-005: Capture Vide - Stockée

**Status:** ✅ ACCEPTÉ

**Context:** Que faire d'une Capture sans todo ni idea extraite ? (ex: "Il fait beau")

**Decision:** Stockée quand même, user décide de la conserver ou supprimer.

---

## Rationale

- Principe NFR6 : "0 capture perdue, jamais"
- Contexte futur potentiel (pensée spontanée peut devenir pertinente plus tard)
- Stockage texte minime (quelques octets)
- User autonomie : UI peut filtrer "Avec actions/idées uniquement"

---

## Consequences

**Bénéfices:**
- Historique complet préservé
- Capture reste transcrite et résumée (avec tags potentiels)

**UX Impact:**
- UI doit permettre filtrage pour ne pas polluer le feed
- Options: "Afficher tout" vs "Avec actions/idées uniquement"

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
