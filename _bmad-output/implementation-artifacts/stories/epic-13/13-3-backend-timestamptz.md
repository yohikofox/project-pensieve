# Story 13.3: Corriger les Types de Colonnes Date vers TIMESTAMPTZ

Status: ready-for-dev

## Story

As a **developer**,
I want **all date columns in backend entities to use `TIMESTAMPTZ` (timestamp with time zone)**,
So that **no timezone information is lost when storing dates, complying with ADR-026 R5**.

## Context

Audit ADR-026 (2026-02-17) révèle :
- `capture.entity.ts:75-78` : `@CreateDateColumn()` et `@UpdateDateColumn()` sans `{ type: 'timestamptz' }`
- TypeORM génère par défaut `timestamp without time zone` → perte d'information timezone
- `capture-state.entity.ts` : FKs en `number` (`typeId?: number`) — à corriger vers UUID

**Note** : Si la Story 12.1 (BaseEntity) est implémentée correctement avec `{ type: 'timestamptz' }` sur tous les `@CreateDateColumn` / `@UpdateDateColumn` / `@DeleteDateColumn`, cette story vérifie la complétude et corrige les cas résiduels non couverts par la BaseEntity.

## Acceptance Criteria

### AC1: `@CreateDateColumn` avec `{ type: 'timestamptz' }` sur toutes les entités
**Given** entities declare `@CreateDateColumn()` without type
**When** I update via BaseEntity inheritance (story 12.1) or direct fix
**Then** all `created_at` columns in PostgreSQL are of type `TIMESTAMPTZ`
**And** no `TIMESTAMP WITHOUT TIME ZONE` columns exist for date fields

### AC2: `@UpdateDateColumn` avec `{ type: 'timestamptz' }` sur toutes les entités
**Given** entities declare `@UpdateDateColumn()` without type
**When** I update all entity date columns
**Then** all `updated_at` columns in PostgreSQL are of type `TIMESTAMPTZ`

### AC3: `@DeleteDateColumn` avec `{ type: 'timestamptz' }` (soft delete)
**Given** soft delete is implemented (story 12.3)
**When** I verify the DeleteDateColumn declaration
**Then** `deleted_at` columns use `TIMESTAMPTZ NULL`

### AC4: Migration pour colonnes existantes
**Given** existing tables have `TIMESTAMP WITHOUT TIME ZONE` columns
**When** the migration runs
**Then** existing columns are altered to `TIMESTAMPTZ`:
```sql
ALTER TABLE captures
  ALTER COLUMN created_at TYPE TIMESTAMPTZ,
  ALTER COLUMN updated_at TYPE TIMESTAMPTZ;
```
**And** existing data is preserved (PostgreSQL handles implicit conversion)

### AC5: Élimination des FKs entières résiduelles
**Given** `capture.entity.ts` has `typeId?: number` and `stateId?: number` as integer FKs
**When** I migrate these FKs to UUID (aligned with story 12.2)
**Then** all FK columns referencing UUID PKs are themselves `UUID` type (not `integer` or `number`)

### AC6: Validation TypeScript et runtime
**Given** all date columns are `TIMESTAMPTZ`
**When** dates are read from the database
**Then** JavaScript `Date` objects preserve timezone information
**And** no date shifts are observed when reading/writing across timezones

## Tech Notes

- **TypeORM syntax** : `@CreateDateColumn({ type: 'timestamptz' })`, `@UpdateDateColumn({ type: 'timestamptz' })`
- **Si BaseEntity implémentée** (story 12.1 first) : Cette story ne fait que vérifier et corriger les entités qui ne passeraient pas par BaseEntity
- **Migration ALTER** : PostgreSQL convertit implicitement `TIMESTAMP` → `TIMESTAMPTZ` sans perte (assume timezone UTC si non défini)
- **Scope de vérification** : Toutes les tables backend (captures, thoughts, ideas, todos, notifications, authorization tables...)
- **FKs entières** : Ce point est chevauchant avec story 12.2 — coordonner pour éviter double migration

## Related

- Story 12.1: BaseEntity (doit avoir `{ type: 'timestamptz' }` — cette story vérifie)
- Story 12.2: PK UUID (FKs entières → UUID)
- ADR-026 R5: Types canoniques TIMESTAMPTZ

## Definition of Done

- [ ] Audit complet : liste de toutes les colonnes `TIMESTAMP WITHOUT TIME ZONE` (rapport dans devnotes)
- [ ] Toutes les colonnes `created_at`, `updated_at`, `deleted_at` sont `TIMESTAMPTZ`
- [ ] Migration TypeORM ALTER TABLE créée et testée
- [ ] FKs entières résiduelles migrées en UUID
- [ ] Tests unitaires : vérification des types de colonnes via TypeORM metadata
- [ ] Zero régression sur tests existants
- [ ] Vérification manuelle : pas de dérive de timezone sur dates existantes après migration
