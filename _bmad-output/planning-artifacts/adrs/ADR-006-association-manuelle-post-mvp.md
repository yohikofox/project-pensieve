---
adr: ADR-006
title: "Association Manuelle Capture ↔ Todo - Post-MVP"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design - Post-MVP Feature"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-006: Association Manuelle Capture ↔ Todo - Post-MVP

**Status:** ✅ ACCEPTÉ (Post-MVP)

**Context:** User peut-il lier manuellement une Capture à une Todo existante après coup ?

**Decision:** Pas MVP, mais architecture prévue via tags communs.

---

## Use Case Post-MVP

- Todo extraite : "Investiguer solutions compta"
- Plus tard, user capture : "Article intéressant sur pain points compta"
- Tags communs détectés → suggestion de lien

---

## Rationale

- Complexité UX non prioritaire pour MVP
- Tags générés par IA permettront d'implémenter cette fonctionnalité facilement plus tard
- Architecture events-based permet d'ajouter `TodoLinkedToCapture` event en V1.5

---

## Consequences

**MVP:**
- Pas d'association manuelle

**Post-MVP:**
- Via tags ou UI explicite
- Event `TodoLinkedToCapture` prévu dans l'architecture

**Implementation Future:**
```typescript
// Event prévu pour V1.5
class TodoLinkedToCapture extends DomainEvent {
  constructor(
    public readonly todoId: string,
    public readonly captureId: string,
    public readonly linkedBy: 'user' | 'ai-suggestion'
  ) {}
}
```

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
