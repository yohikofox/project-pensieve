# Story 12.4: Supprimer les Cascades TypeORM et Gérer la Suppression en Couche Applicative

Status: backlog

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
**Then** the application service (`ThoughtService` or equivalent) explicitly soft-deletes all related Ideas
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

## Tech Notes

- **Pattern** : Transaction TypeORM avec `QueryRunner` ou `DataSource.transaction()`
- **Service target** : `modules/knowledge/application/services/` — service gérant la suppression de `Thought`
- **Audit des cascades** : Grep sur `cascade: true` dans tout le backend — liste exhaustive à traiter
- **Result Pattern** : La suppression retourne un `Result<void>`, pas de `throw`
- **Log structuré** : `logger.info('thought.deleted', { thoughtId, relatedIdeas: ideas.map(i => i.id) })`

## Related

- Story 12.1: BaseEntity (prérequis)
- Story 12.3: Soft Delete (prérequis — soft-delete des Ideas liées)
- ADR-026 R3: Cascades interdites sans décision documentée

## Definition of Done

- [ ] `cascade: true` retiré de `thought.entity.ts`
- [ ] Service applicatif gère la suppression explicite des Ideas liées
- [ ] Transaction atomique (rollback sur échec)
- [ ] Résultat via Result Pattern (pas de throw)
- [ ] Audit de toutes les cascades restantes dans le backend (rapport dans devnotes)
- [ ] Tests BDD : min 3 scénarios
- [ ] Tests unitaires : transaction rollback, cascade applicative
- [ ] Zero régression sur suite de tests existante
