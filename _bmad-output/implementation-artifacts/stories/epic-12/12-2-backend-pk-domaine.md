# Story 12.2: Remplacer @PrimaryGeneratedColumn par UUID Généré dans le Domaine

Status: ready-for-dev

## Story

As a **developer**,
I want **all backend entity PKs to be UUIDs generated in the domain layer, not by PostgreSQL**,
So that **the domain controls its own identity, enabling offline-first patterns and clean DDD (ADR-026 R1)**.

## Context

Audit ADR-026 (2026-02-17) révèle des violations `@PrimaryGeneratedColumn('uuid')` dans toutes les entités backend :
- `capture.entity.ts:34`
- `thought.entity.ts:23`
- `capture-state.entity.ts:9` (doublement interdit : entier + DB-generated)

En délégant la génération des UUIDs à PostgreSQL, le domaine perd le contrôle de son identité — incompatible avec l'offline-first et le DDD strict.

**Prérequis** : Story 12.1 doit être complétée (BaseEntity avec `@PrimaryColumn('uuid')` déjà défini).

## Acceptance Criteria

### AC1: Remplacement dans toutes les entités principales
**Given** all backend entities use `@PrimaryGeneratedColumn`
**When** I migrate to `@PrimaryColumn('uuid')` via BaseEntity inheritance
**Then** the following entities no longer declare their own `id` column:
- `capture.entity.ts`
- `thought.entity.ts`
- `idea.entity.ts`
- `todo.entity.ts`
- `capture-state.entity.ts` (migration vers UUID depuis integer)

### AC2: UUID généré dans la couche domaine ou applicative
**Given** entities have `@PrimaryColumn('uuid')`
**When** a new entity is created
**Then** the UUID is generated BEFORE persisting (in service or domain factory):
```typescript
// ✅ Correct: UUID generated in application layer
import { v7 as uuidv7 } from 'uuid';

const capture = new CaptureEntity();
capture.id = uuidv7();
```
**And** no entity has a DB-level `DEFAULT gen_random_uuid()`

### AC3: Migration TypeORM correcte
**Given** existing data in the database with old PKs
**When** the migration runs
**Then** existing rows preserve their UUIDs (no data loss)
**And** the PK column type is `UUID NOT NULL PRIMARY KEY` (without DB default)
**And** foreign keys referencing these PKs are updated accordingly

### AC4: Élimination des FKs entières
**Given** `capture-state.entity.ts` used integer PK and FKs (`typeId?: number`, `stateId?: number`)
**When** migration completes
**Then** all FKs referencing states/types use UUID
**And** existing data is migrated with proper UUID mapping

### AC5: Tests de non-régression
**Given** all entity PKs are migrated
**When** I run the full test suite
**Then** all existing unit tests and BDD tests pass without modification
**And** new unit tests validate UUID generation in application layer

## Tech Notes

- **UUID version** : Préférer `uuid` v7 (time-ordered, better PostgreSQL index performance) ou `crypto.randomUUID()` (Node 22 natif)
- **Migration strategy** : Backup obligatoire avant toute migration en prod. Utiliser transactions.
- **capture-state.entity.ts** : migration integer → UUID nécessite une stratégie de mapping (créer une table de mapping temporaire ou recréer les données)
- **FKs concernées** : `typeId`, `stateId` sur `capture.entity.ts` → migrer vers `type_id UUID` et `state_id UUID`
- **NEVER** `synchronize: true` — toujours via `npm run migration:generate` puis `migration:run`

## Related

- Story 12.1: Créer la BaseEntity Partagée Backend (prérequis)
- Story 12.3: Implémenter soft delete sur toutes les entités
- ADR-026 R1: PKs générées par le domaine

## Definition of Done

- [ ] `@PrimaryGeneratedColumn` absent de toute entité backend
- [ ] Toutes les entités héritent de `BaseEntity` (story 12.1)
- [ ] UUID généré dans la couche applicative (service ou factory)
- [ ] Migration TypeORM créée et testée en local
- [ ] FKs entières (`typeId`, `stateId`) migrées en UUID
- [ ] Tests unitaires : UUID generation, no DB default
- [ ] Tests BDD : création d'entités via API endpoints
- [ ] Zero régression sur suite de tests existante
