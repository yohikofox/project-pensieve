---
adr: ADR-002
title: "Normalization - Domain Service (pas Aggregate)"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-002: Normalization - Domain Service (pas Aggregate)

**Status:** ✅ ACCEPTÉ

**Context:** La normalisation (Whisper, OCR, web scraper) doit-elle être un Aggregate ou un Service ?

**Decision:** Domain Services stateless, pas d'Aggregate `Normalization`.

---

## Rationale

- Transformation technique transitoire
- État métier pertinent reste dans `Capture.state`
- Traçabilité via logs applicatifs suffisante pour MVP
- Pas de business logic complexe dans la normalisation elle-même

---

## Consequences

**Bénéfices:**
- Simplicité : pas d'aggregate supplémentaire
- Pas de double état (Capture + Normalization)

**Future Evolution:**
- Si besoin analytics plus tard → ajouter `NormalizationAttempt` comme Value Object

**Impact:**
- Normalisation reste dans Bounded Context "Normalization"
- Services stateless (WhisperService, OCRService, WebScraperService)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
