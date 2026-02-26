# Architecture Decision Records (ADRs)

Ce dossier contient tous les ADRs (Architecture Decision Records) du projet Pensieve.

## Index des ADRs

| ADR | Titre | Status | Date | Participants |
|-----|-------|--------|------|--------------|
| [ADR-001](./ADR-001-aggregate-granularity.md) | Aggregate Granularity - Aggregates Séparés | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-002](./ADR-002-normalization-domain-service.md) | Normalization - Domain Service | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-003](./ADR-003-sync-infrastructure.md) | Sync - Infrastructure | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-004](./ADR-004-digestion-ia-single-llm-call.md) | Digestion IA - Un seul appel LLM | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-005](./ADR-005-capture-vide-stockee.md) | Capture Vide - Stockée | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-006](./ADR-006-association-manuelle-post-mvp.md) | Association Manuelle Capture ↔ Todo | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-007](./ADR-007-from-scratch-approach.md) | From Scratch Approach | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-008](./ADR-008-anti-corruption-layer.md) | Anti-Corruption Layer (ACL) | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-009](./ADR-009-sync-patterns.md) | Sync Patterns | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-010](./ADR-010-security-encryption.md) | Security & Encryption | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-011](./ADR-011-performance-optimization.md) | Performance Optimization | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-012](./ADR-012-queue-management.md) | Queue Management RabbitMQ | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-013](./ADR-013-notification-system.md) | Notification System | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-014](./ADR-014-storage-management.md) | Storage Management | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-015](./ADR-015-observability-strategy.md) | Observability Strategy | ✅ Accepted | 2026-01-19 | yohikofox, Winston |
| [ADR-016](./ADR-016-hybrid-architecture.md) | Hybrid Architecture - Cloud Auth + Homelab Storage | 🔄 Partiellement Superseded (Auth→ADR-029, Email→ADR-030, Storage✅) | 2026-01-19 | yohikofox, Winston |
| [ADR-017](./ADR-017-ioc-di-strategy.md) | Dependency Injection & IoC Container Strategy | ✅ Accepted | 2026-01-22 | yohikofox, Winston, Amelia, Murat |
| [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md) | Migration WatermelonDB → OP-SQLite | ✅ Accepted | 2026-01-22 | yohikofox, Winston, Amelia |
| [ADR-019](./ADR-019-eventbus-domain-events.md) | EventBus Architecture - Domain Events avec RxJS | ✅ Accepted | 2026-01-24 | yohikofox, Winston, Amelia |
| [ADR-020](./ADR-020-background-processing-strategy.md) | Background Processing Strategy - expo-task-manager | ✅ Accepted | 2026-01-24 | yohikofox, Winston, Amelia |
| [ADR-021](./ADR-021-di-lifecycle-transient-first.md) | DI Lifecycle Strategy - Transient First (Révision ADR-017) | ✅ Accepted | 2026-01-24 | yohikofox, Winston, Amelia |
| [ADR-022](./ADR-022-state-persistence-opsqlite.md) | State Persistence Strategy - OP-SQLite for All State | ✅ Accepted | 2026-01-24 | yohikofox, Winston, Amelia |
| [ADR-023](./ADR-023-error-handling-strategy.md) | Stratégie Unifiée de Gestion des Erreurs - Result Pattern | ✅ Accepted | 2026-02-15 | yohikofox, Winston |
| [ADR-024](./ADR-024-clean-code-standards.md) | Standards Clean Code Appliqués au Projet Pensieve | ✅ Accepted | 2026-02-15 | yohikofox, Winston |
| [ADR-025](./ADR-025-http-client-strategy.md) | HTTP Client Strategy - fetch natif + wrapper custom | ✅ Accepted | 2026-02-15 | yohikofox, Winston, Amelia |
| [ADR-026](./ADR-026-backend-data-model-design-rules.md) | Backend Data Model Design Rules | ✅ Accepted | 2026-02-17 | yohikofox, Winston |
| [ADR-027](./ADR-027-unit-cache-strategy.md) | Unit Cache Strategy — Cache Unitaire Opt-in par Héritage de Repository | ✅ Accepted | 2026-02-18 | yohikofox, Winston |
| [ADR-028](./ADR-028-typescript-type-safety-policy.md) | TypeScript Type Safety Policy — Interdiction de `any`, Hiérarchie de Typage | ✅ Accepted | 2026-02-18 | yohikofox, Winston |

| [ADR-029](./ADR-029-auth-provider-better-auth.md) | Authentication Provider — Better Auth Self-Hosted (Révision Auth ADR-016) | ✅ Accepted | 2026-02-18 | yohikofox, Winston |
| [ADR-030](./ADR-030-transactional-email-provider.md) | Transactional Email Provider — Resend | ✅ Accepted | 2026-02-18 | yohikofox, Winston |
| [ADR-031](./ADR-031-rich-domain-entities-mobile.md) | Rich Domain Entities — Classes avec Invariants pour le Mobile | ✅ Accepted | 2026-02-19 | yohikofox, Winston |

**Total:** 31 ADRs documentés (40+ sous-décisions architecturales)

---

## Format ADR Standard

Chaque ADR suit le format suivant :

```markdown
---
adr: ADR-XXX
title: "Titre de la décision"
date: YYYY-MM-DD
status: "✅ Accepted | ⏳ Proposed | ❌ Rejected | 🔄 Superseded"
supersedes: "ADR-YYY (optionnel)"
superseded_by: "ADR-ZZZ (optionnel)"
context: "Story X.Y - Contexte d'origine"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
  - Amelia (Dev Agent)
---

# ADR-XXX: Titre de la Décision

**Date:** YYYY-MM-DD
**Status:** ✅ Accepted
**Context:** Story X.Y - Description du contexte
**Decision Makers:** Liste des participants

---

## Context & Problem

**Problème à résoudre :**

Description détaillée du problème ou du besoin qui a motivé cette décision architecturale.

**Contraintes identifiées :**
- Contrainte 1
- Contrainte 2

**Analogie :** (si pertinent)

---

## Decision

**Décision prise :**

Description claire et concise de la décision architecturale.

**Options évaluées :**

| Critère | Option A | Option B | Option C | Gagnant |
|---------|----------|----------|----------|---------|
| Critère 1 | ... | ... | ... | ... |
| Score final | X/10 | Y/10 | Z/10 | Option A |

**Rationale :**

Justification détaillée de pourquoi cette option a été choisie.

---

## Consequences

**✅ Bénéfices :**
1. Bénéfice 1
2. Bénéfice 2

**⚠️ Trade-offs acceptés :**
1. Trade-off 1 avec mitigation
2. Trade-off 2 avec mitigation

**🔄 Impact sur architecture existante :**
- Impact 1
- Impact 2

---

## Implementation

**Étapes de mise en œuvre :**
1. Étape 1
2. Étape 2

**Files Modified :**
```
path/to/file1
path/to/file2
```

**Effort réel :** X heures/jours

---

## Validation Criteria

ADR considéré succès SI :
- ✅ Critère 1 (validé YYYY-MM-DD)
- ⏳ Critère 2 (en cours)

**Review Date :** YYYY-MM (date de revue post-implémentation)

---

## References

- Lien 1
- Lien 2

---

## Decision Log

**YYYY-MM-DD** - Discussion participants

→ Timeline de la décision
→ Trade-offs discutés
→ Décision finale

**Participants :**
- Personne 1 (Rôle)
- Personne 2 (Rôle)

---
```

---

## Quand créer un nouvel ADR ?

Créez un nouvel ADR quand :

1. **Choix technologique majeur** : Framework, bibliothèque, base de données
2. **Pattern architectural** : DDD, CQRS, Event Sourcing, microservices
3. **Décision impactante** : Sync strategy, security model, deployment strategy
4. **Migration/Changement** : Remplacement d'une technologie, refactoring majeur
5. **Trade-off significatif** : Quand vous acceptez des compromis importants

**Ne créez PAS d'ADR pour :**
- Choix d'implémentation mineurs (ex: nom de variable)
- Décisions réversibles facilement (ex: couleur UI)
- Décisions évidentes sans alternatives

---

## Process de création ADR

1. **Discuter** le problème avec l'équipe (yohikofox, Winston, agents concernés)
2. **Créer le fichier** `ADR-XXX-titre-en-kebab-case.md`
3. **Remplir le template** avec contexte, décision, conséquences
4. **Valider** avec l'équipe avant status "✅ Accepted"
5. **Mettre à jour cet index** (README.md)
6. **Référencer** dans architecture.md si pertinent

---

## Architecture Decisions vs Implementation Decisions

**Architecture Decisions (ADR)** :
- Impact plusieurs stories/epics
- Coût élevé de changement (> 1 semaine)
- Implications long terme (> 6 mois)
- Nécessite consensus équipe
- **Exemple :** ADR-018 Migration WatermelonDB → OP-SQLite

**Implementation Decisions (Story notes)** :
- Scope limité à une story
- Coût faible de changement (< 1 jour)
- Impact court terme
- Décision autonome dev
- **Exemple :** Choix d'un nom de fonction, pattern de validation local

---

## Lifecycle ADR

```
Proposed (⏳) → Accepted (✅) → [Review après 1-3 mois] → Maintenu OU Superseded (🔄)
                              ↘ Rejected (❌)
```

**Status possibles :**
- **⏳ Proposed** : En discussion, pas encore validé
- **✅ Accepted** : Validé et en cours d'implémentation/implémenté
- **❌ Rejected** : Rejeté après discussion (garder trace du "pourquoi")
- **🔄 Superseded** : Remplacé par un nouvel ADR (lien vers nouveau)

---

## Maintenance

- **Review périodique** : Tous les 3 mois, vérifier si ADRs sont toujours pertinents
- **Update si nécessaire** : Ajouter section "Amendment" si contexte change
- **Supersede si obsolète** : Créer nouvel ADR et marquer ancien comme 🔄 Superseded

---

| [ADR-032](./ADR-032-action-bc-rename-task-bc.md) | Renommage Action BC → Task BC — Vocabulaire DDD précis | ✅ Accepted | 2026-02-26 | yohikofox, Winston |
| [ADR-033](./ADR-033-delegation-bounded-context.md) | Nouveau BC Delegation — Frontières, ExecutionJob, intégrations tierces | ✅ Accepted | 2026-02-26 | yohikofox, Winston |
| [ADR-034](./ADR-034-action-todo-executionjob-semantic-model.md) | Modèle sémantique Action/Todo/ExecutionJob — Pipeline de délégation | ✅ Accepted | 2026-02-26 | yohikofox, Winston |
| [ADR-035](./ADR-035-delegation-choreography-option-a.md) | Chorégraphie CaptureDigested — Option A event-driven (vs orchestration) | ✅ Accepted | 2026-02-26 | yohikofox, Winston |
| [ADR-036](./ADR-036-llm-local-first-byok.md) | LLM Local-First Strategy — Contrainte économique pré-revenue + BYOK optionnel | ✅ Accepted | 2026-02-26 | yohikofox, Winston |

---

**Dernière mise à jour :** 2026-02-26 (ADR-032 à ADR-036 ajoutés — Session Architecture Mode Délégation)
**Maintenu par :** Winston (Architect) + yohikofox (Product Owner)
