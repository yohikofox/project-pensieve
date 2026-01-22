---
adr: ADR-001
title: "Aggregate Granularity - Aggregates Séparés"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-001: Aggregate Granularity - Aggregates Séparés

**Status:** ✅ ACCEPTÉ

**Context:** Décider si les phases (Capture, Thought, Idea, Project) sont un seul aggregate ou plusieurs.

**Decision:** Aggregates séparés avec communication via Domain Events.

---

## Rationale

- `Capture` → cycle de vie distinct (normalisation peut échouer indépendamment)
- `Thought` → cycle de vie distinct (digestion complète même si rien n'est extrait)
- `Idea` → cycle de vie long (concordance, germination sur plusieurs semaines/mois)
- `Project` → cycle de vie long et distinct (agrège plusieurs Ideas)
- Frontières transactionnelles naturelles

---

## Consequences

**Bénéfices:**
- Loose coupling entre phases
- Communication asynchrone via Domain Events
- Résilience : échec d'une phase n'impacte pas les autres

**Impact:**
- Architecture event-driven nécessaire
- Chaque aggregate a son propre repository
- Transactions isolées par aggregate

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
