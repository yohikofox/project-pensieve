---
adr: ADR-003
title: "Sync - Infrastructure (pas Bounded Context)"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-003: Sync - Infrastructure (pas Bounded Context)

**Status:** ✅ ACCEPTÉ

**Context:** Le sync offline-first est-il un contexte métier ou de l'infrastructure ?

**Decision:** Infrastructure pure, pas de Bounded Context métier.

---

## Rationale

- ~~WatermelonDB gère déjà le sync protocol~~
- ⚠️ **UPDATE (2026-01-22):** OP-SQLite remplace WatermelonDB - sync protocol sera implémenté manuellement Epic 6. Voir [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md)
- Pattern générique offline-first (pas spécifique à Pensine)
- Résolution conflits simple (last-write-wins ou timestamp)
- Pas de logique métier Pensine dans le sync

---

## Consequences

**Bénéfices:**
- Sync = infrastructure technique
- Domain Events doivent être syncables (transports par le sync)
- Pas de SyncContext dans le modèle métier

**Impact:**
- Stratégie sync définie dans [ADR-009](./ADR-009-sync-patterns.md)
- Implémentation Epic 6 avec OP-SQLite (manuel)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
