# Story 12.3: Implémenter le Soft Delete sur Toutes les Entités Backend

Status: ready-for-dev

## Story

As a **developer**,
I want **all backend entities to use soft delete (`deletedAt`) instead of hard delete or status flags**,
So that **data is never lost, audit trails are preserved, and the system complies with ADR-026 R4**.

## Context

Audit ADR-026 (2026-02-17) révèle l'absence totale de `@DeleteDateColumn` sur les entités backend. Actuellement :
- `capture.entity.ts` : pas de `deletedAt`
- `thought.entity.ts` : utilise `_status: 'active' | 'deleted'` (texte libre) au lieu de `deletedAt`
- Aucune entité ne possède `@DeleteDateColumn({ type: 'timestamptz' })`

Le soft delete est déjà dans `BaseEntity` (story 12.1). Cette story migre les logiques de suppression existantes vers ce mécanisme.

**Prérequis** : Stories 12.1 et 12.2 complétées.

## Acceptance Criteria

### AC1: `deletedAt` présent via BaseEntity sur toutes les entités
**Given** all entities inherit from BaseEntity (story 12.1)
**When** the migration runs
**Then** all entity tables have a `deleted_at TIMESTAMPTZ NULL` column
**And** TypeORM `@DeleteDateColumn` is active (enables automatic filtering)

### AC2: Suppression du champ `_status` textuel dans `thought.entity.ts`
**Given** `thought.entity.ts` uses `_status: 'active' | 'deleted'` as text column
**When** I migrate to soft delete
**Then** the `_status` column is removed from the entity
**And** a migration updates existing 'deleted' rows by setting `deleted_at = updated_at` (or NOW())
**And** 'active' rows keep `deleted_at = NULL`

### AC3: Repositories utilisent `softDelete()` et `withDeleted()`
**Given** soft delete is enabled on all entities
**When** a delete operation is called
**Then** repositories use TypeORM `softDelete(id)` instead of `delete(id)`
**And** standard queries automatically exclude soft-deleted records (`deleted_at IS NULL`)
**And** admin/audit queries can use `withDeleted()` to include soft-deleted records

### AC4: API endpoints respectent le soft delete
**Given** soft delete is implemented
**When** a client calls DELETE /captures/:id or similar endpoints
**Then** the record is soft-deleted (deleted_at set), not hard-deleted
**And** subsequent GET requests for the same resource return 404 (filtered by TypeORM)
**And** the soft-deleted record is still visible in admin queries

### AC5: Tests BDD pour le soft delete
**Given** soft delete is implemented
**When** I run acceptance tests
**Then** there are BDD scenarios validating:
- Deleted entity is not returned in standard list queries
- Deleted entity can be retrieved with admin/audit access
- Re-querying a soft-deleted ID returns 404

## Tech Notes

- **TypeORM Soft Delete** : `@DeleteDateColumn` + activer `{ softDelete: true }` dans `TypeOrmModule` options ou utiliser explicitement `repository.softDelete(id)`
- **Migration thought_status** : `UPDATE thoughts SET deleted_at = updated_at WHERE _status = 'deleted'; ALTER TABLE thoughts DROP COLUMN _status;`
- **thought_statuses table** : La migration de `_status` vers table référentielle est couverte par Story 13.2 (priorité moyenne). Cette story supprime juste le champ textuel sur `thought`.
- **Scope** : Toutes les entités backend héritant de BaseEntity (capture, thought, idea, todo, notification, admin_user...)
- **RGPD** : Le soft delete respecte RGPD — l'anonymisation reste gérée par `RgpdService`

## Related

- Story 12.1: Créer la BaseEntity Partagée Backend (prérequis)
- Story 12.2: Remplacer @PrimaryGeneratedColumn (prérequis)
- Story 13.2: Tables référentielles pour les statuts (complémentaire)
- ADR-026 R4: Soft Delete obligatoire

## Definition of Done

- [ ] `deleted_at TIMESTAMPTZ NULL` présent sur toutes les tables backend
- [ ] Champ `_status` retiré de `thought.entity.ts`
- [ ] Migration de données : `_status = 'deleted'` → `deleted_at` renseigné
- [ ] Repositories utilisent `softDelete()` / `withDeleted()`
- [ ] Endpoints DELETE vérifié : comportement soft delete confirmé
- [ ] Tests BDD : min 3 scénarios (suppression, vérification 404, audit access)
- [ ] Tests unitaires repositories : soft delete appliqué
- [ ] Zero régression sur suite de tests existante
