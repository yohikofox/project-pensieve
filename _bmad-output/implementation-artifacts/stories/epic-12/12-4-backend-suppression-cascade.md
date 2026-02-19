# Story 12.4: Supprimer les Cascades TypeORM et Gérer la Suppression en Couche Applicative

Status: done

## Story

As a **developer**,
I want **cascade delete to be managed explicitly in the application layer, not delegated to TypeORM**,
So that **deletion behavior is explicit, auditable, and follows documented business decisions (ADR-026 R3)**.

## Context

Audit ADR-026 (2026-02-17) révèle :
- `thought.entity.ts:55` : `@OneToMany(() => Idea, (idea) => idea.thought, { cascade: true })`
- Cette cascade n'est pas documentée comme décision architecturale explicite

La cascade TypeORM est dangereuse car :
- Elle opère au niveau base de données, contournant la couche applicative
- Elle empêche d'appliquer le soft delete correctement
- Elle crée des suppressions silencieuses non tracées dans les logs

**Prérequis** : Stories 12.1 et 12.3 complétées (soft delete en place).

## Acceptance Criteria

### AC1: Suppression de `cascade: true` sur la relation Thought → Idea
**Given** `thought.entity.ts` has `{ cascade: true }` on `ideas` relation
**When** I remove the cascade option
**Then** `thought.entity.ts:55` no longer has `{ cascade: true }`
**And** the TypeORM relation is declared without cascade: `@OneToMany(() => Idea, (idea) => idea.thought)`

### AC2: Service applicatif gérant la suppression des Idées liées
**Given** cascade is removed from the entity
**When** a Thought is soft-deleted
**Then** the application service (`ThoughtDeleteService`) explicitly soft-deletes all related Ideas
**And** the soft-deletion of Ideas is logged (structured log with `idea.id`, `thought.id`, `reason`)
**And** the operation is atomic (transaction wrapping both soft-deletes)

### AC3: Transaction atomique
**Given** a Thought deletion that cascades to Ideas
**When** the soft-delete of any related Idea fails
**Then** the entire transaction is rolled back (the Thought is not deleted either)
**And** an error is returned to the caller (Result Pattern — no throw)

### AC4: Audit des cascades restantes
**Given** cascade: true has been removed from Thought → Idea
**When** I audit all other entity relations
**Then** no other `{ cascade: true }` relation exists without an explicit documented decision
**And** if other cascades are found, they are either removed or documented with a comment referencing the ADR decision

### AC5: Tests BDD pour la suppression explicite
**Given** cascade is managed in the application layer
**When** a Thought is soft-deleted
**Then** BDD scenarios validate:
- Related Ideas are also soft-deleted
- Transaction rollback on failure
- Soft-deleted Ideas are not returned in standard queries

## Tasks/Subtasks

- [x] **Task 1: Retirer `cascade: true` de `thought.entity.ts`** (AC1)
  - [x] 1.1 Modifier `thought.entity.ts:41` — retirer `{ cascade: true }` de la relation `ideas`
  - [x] 1.2 Documenter `onDelete: 'CASCADE'` dans `idea.entity.ts` (contrainte FK SQL — décision ADR-026 R3 documentée)

- [x] **Task 2: Créer `Result<T>` type pour le backend** (AC3)
  - [x] 2.1 Créer `src/common/types/result.type.ts` avec `success()` et `transactionError()`

- [x] **Task 3: Créer `ThoughtDeleteService`** (AC2, AC3)
  - [x] 3.1 Créer `src/modules/knowledge/application/services/thought-delete.service.ts`
  - [x] 3.2 Implémenter `softDeleteWithRelated(thoughtId): Promise<Result<void>>` avec transaction atomique
  - [x] 3.3 Log structuré des suppressions (thoughtId + ideaIds)

- [x] **Task 4: Mettre à jour `ThoughtsController`** (AC2)
  - [x] 4.1 Injecter `ThoughtDeleteService` dans `ThoughtsController`
  - [x] 4.2 Utiliser `ThoughtDeleteService.softDeleteWithRelated()` dans `deleteThought()`

- [x] **Task 5: Enregistrer `ThoughtDeleteService` dans `KnowledgeModule`** (AC2)

- [x] **Task 6: Tests unitaires** (AC3)
  - [x] 6.1 Créer `thought-delete.service.spec.ts` — transaction rollback, cascade applicative

- [x] **Task 7: Tests BDD** (AC5)
  - [x] 7.1 Créer `test/acceptance/features/story-12-4-backend-suppression-cascade.feature` (3 scénarios)
  - [x] 7.2 Créer `test/acceptance/story-12-4.test.ts` (step definitions)

- [x] **Task 8: Validation** (AC4)
  - [x] 8.1 Audit complet de toutes les cascades dans le backend (rapport ci-dessous dans Dev Notes)
  - [x] 8.2 Exécuter tests unitaires + acceptance — zéro régression

## Tech Notes

- **Pattern** : Transaction TypeORM avec `DataSource.transaction()`
- **Service target** : `modules/knowledge/application/services/thought-delete.service.ts`
- **Audit des cascades** : Grep sur `cascade: true` dans tout le backend — liste exhaustive à traiter
- **Result Pattern** : La suppression retourne un `Result<void>`, pas de `throw`
- **Log structuré** : `logger.log('thought.deleted', { thoughtId, relatedIdeas: ideas.map(i => i.id) })`

## Related

- Story 12.1: BaseEntity (prérequis)
- Story 12.3: Soft Delete (prérequis — soft-delete des Ideas liées)
- ADR-026 R3: Cascades interdites sans décision documentée

## Definition of Done

- [x] `cascade: true` retiré de `thought.entity.ts`
- [x] Service applicatif gère la suppression explicite des Ideas liées
- [x] Transaction atomique (rollback sur échec)
- [x] Résultat via Result Pattern (pas de throw)
- [x] Audit de toutes les cascades restantes dans le backend (rapport dans devnotes)
- [x] Tests BDD : min 3 scénarios
- [x] Tests unitaires : transaction rollback, cascade applicative
- [x] Zero régression sur suite de tests existante

## Dev Notes

### Audit des Cascades Backend (2026-02-18)

**Cascades TypeORM `cascade: true` trouvées :**
- `thought.entity.ts:41` → **RETIRÉ** dans cette story (AC1 satisfait)

**Contraintes FK SQL `onDelete: 'CASCADE'` trouvées :**
- `idea.entity.ts:38` → `@ManyToOne(() => Thought, ..., { onDelete: 'CASCADE' })`
  - **Décision** : CONSERVÉ — Cette contrainte FK SQL est différente du cascade ORM TypeORM.
  - **Rationale ADR-026 R3** : Cette contrainte est intentionnelle pour garantir l'intégrité référentielle lors d'opérations de maintenance DB directe (hard-delete d'entités de test, cleanup admin). Avec le soft delete TypeORM (`softDelete()`), cette contrainte n'est jamais déclenchée car softDelete émet un UPDATE (pas un DELETE SQL). Documenté ici comme décision explicite conformément à ADR-026 R3.

- `todo.entity.ts:71` → `@ManyToOne(() => Thought, { onDelete: 'CASCADE' })`
  - **Décision** : CONSERVÉ — Même rationale que `idea.entity.ts:38`.
  - **Rationale ADR-026 R3** : Contrainte FK SQL uniquement, jamais déclenchée par `softDelete()` (UPDATE). Garantit l'intégrité référentielle lors de suppressions DB directes (maintenance admin). Documenté via commentaire ADR-026 R3 dans `todo.entity.ts` lors de la code review story 12.4.
  - **Note** : `ThoughtDeleteService` gère le soft-delete des Ideas liées mais pas des Todos. Le scope de cette story est limité à la relation Thought → Idea (seule relation qui avait un `cascade: true` ORM). Les Todos ne peuvent pas être soft-deleted de façon automatique sans une story dédiée (Thought → Todos → cascade applicative).

**Migration** : Aucune migration SQL nécessaire — la suppression du `cascade: true` TypeORM n'impacte pas le schéma DB. C'était une configuration ORM uniquement.

### Plan d'implémentation

1. `ThoughtDeleteService` utilise `DataSource.transaction()` pour atomicité
2. Le service cherche les Ideas non-supprimées liées au Thought, les soft-delete, puis soft-delete le Thought
3. En cas d'erreur, la transaction rollback et le service retourne `Result<void>` avec type `'transaction_error'`
4. Le `ThoughtsController` mappe `transaction_error` → 500 InternalServerErrorException
5. Le controller reste inchangé pour les succès (204 No Content)

## Dev Agent Record

### Implementation Plan

Story 12.4 : Supprimer les Cascades TypeORM — gestion applicative des suppressions.

**Fichiers créés :**
- `pensieve/backend/src/common/types/result.type.ts`
- `pensieve/backend/src/modules/knowledge/application/services/thought-delete.service.ts`
- `pensieve/backend/src/modules/knowledge/application/services/thought-delete.service.spec.ts`
- `pensieve/backend/test/acceptance/features/story-12-4-backend-suppression-cascade.feature`
- `pensieve/backend/test/acceptance/story-12-4.test.ts`

**Fichiers modifiés :**
- `pensieve/backend/src/modules/knowledge/domain/entities/thought.entity.ts` (retrait `cascade: true`)
- `pensieve/backend/src/modules/knowledge/domain/entities/idea.entity.ts` (ajout commentaire ADR-026 R3)
- `pensieve/backend/src/modules/knowledge/application/controllers/thoughts.controller.ts` (injection ThoughtDeleteService)
- `pensieve/backend/src/modules/knowledge/knowledge.module.ts` (ajout provider)

### Completion Notes

- ✅ AC1: `cascade: true` retiré de `thought.entity.ts`
- ✅ AC2: `ThoughtDeleteService.softDeleteWithRelated()` implémenté avec log structuré
- ✅ AC3: Transaction `DataSource.transaction()` — rollback automatique sur exception
- ✅ AC4: Audit complet — seule la FK SQL `onDelete: 'CASCADE'` dans `idea.entity.ts` reste, documentée comme décision explicite ADR-026 R3
- ✅ AC5: 3 scénarios BDD + tests unitaires créés

## File List

- `pensieve/backend/src/common/types/result.type.ts` (créé)
- `pensieve/backend/src/modules/knowledge/domain/entities/thought.entity.ts` (modifié)
- `pensieve/backend/src/modules/knowledge/domain/entities/idea.entity.ts` (modifié)
- `pensieve/backend/src/modules/knowledge/application/services/thought-delete.service.ts` (créé)
- `pensieve/backend/src/modules/knowledge/application/services/thought-delete.service.spec.ts` (créé)
- `pensieve/backend/src/modules/knowledge/application/controllers/thoughts.controller.ts` (modifié)
- `pensieve/backend/src/modules/knowledge/knowledge.module.ts` (modifié)
- `pensieve/backend/test/acceptance/features/story-12-4-backend-suppression-cascade.feature` (créé)
- `pensieve/backend/test/acceptance/story-12-4.test.ts` (créé)
- `pensieve/backend/src/modules/action/domain/entities/todo.entity.ts` (modifié — code review: documentation ADR-026 R3)

## Change Log

- 2026-02-18: Story 12.4 implémentée — cascade TypeORM retirée, ThoughtDeleteService créé, tests BDD et unitaires ajoutés
