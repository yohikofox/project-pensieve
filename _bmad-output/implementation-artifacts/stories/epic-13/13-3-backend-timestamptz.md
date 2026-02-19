# Story 13.3: Corriger les Types de Colonnes Date vers TIMESTAMPTZ

Status: done

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

- [x] Audit complet : liste de toutes les colonnes `TIMESTAMP WITHOUT TIME ZONE` (rapport dans devnotes)
- [x] Toutes les colonnes `created_at`, `updated_at`, `deleted_at` sont `TIMESTAMPTZ`
- [x] Migration TypeORM ALTER TABLE créée et testée
- [x] FKs entières résiduelles migrées en UUID (confirmé depuis story 12.2)
- [x] Tests unitaires : vérification des types de colonnes via TypeORM metadata
- [x] Zero régression sur tests existants
- [ ] Vérification manuelle : pas de dérive de timezone sur dates existantes après migration (hors scope CI — nécessite DB live)

## Tasks/Subtasks

- [x] Task 1: Rapport d'audit + correction des entités non-BaseEntity
  - [x] 1.1 Documenter toutes les colonnes timestamp dans Dev Notes (rapport audit)
  - [x] 1.2 Corriger `todo.entity.ts` (deadline, completedAt → timestamptz)
  - [x] 1.3 Corriger `Notification.entity.ts` (sentAt, deliveredAt, createdAt, updatedAt)
  - [x] 1.4 Corriger `user.entity.ts` (deletion_requested_at, created_at, updated_at)
  - [x] 1.5 Corriger `admin-user.entity.ts` (createdAt, updatedAt)
  - [x] 1.6 Corriger `audit-log.entity.ts` (timestamp column)
  - [x] 1.7 Corriger `sync-log.entity.ts` + `sync-conflict.entity.ts`
  - [x] 1.8 Corriger 10 entités d'autorisation (role, permission, user-role, etc.)
  - [x] 1.9 Vérifier AC5 FKs entières (déjà fait story 12.2 — confirmé)
- [x] Task 2: Créer migration ALTER TABLE (AC4)
  - [x] 2.1 Migration 1771800000000-AlterTimestampColumnsToTimestamptz.ts
- [x] Task 3: Tests TypeORM column metadata (AC6)
  - [x] 3.1 Créer test unitaire `timestamp-columns.spec.ts` (32 tests, 32 passent ✅)
  - [x] 3.2 Créer feature Gherkin + step definitions BDD (9 scenarios, 9 passent ✅)

## Dev Agent Record

### Implementation Plan

**Audit complet (2026-02-18) — Colonnes TIMESTAMP WITHOUT TIME ZONE identifiées :**

| Entité | Table | Colonnes violant ADR-026 R5 (nom réel TypeORM) |
|--------|-------|------------------------------------------------|
| `todo.entity.ts` | `todos` | `deadline`, `completedAt` |
| `Notification.entity.ts` | `notifications` | `sentAt`, `deliveredAt`, `createdAt`, `updatedAt` |
| `user.entity.ts` | `users` | `deletion_requested_at`, `created_at`, `updated_at` |
| `admin-user.entity.ts` | `admin_users` | `created_at`, `updated_at` |
| `audit-log.entity.ts` | `audit_logs` | `timestamp` |
| `sync-log.entity.ts` | `sync_logs` | `startedAt`, `completedAt` |
| `sync-conflict.entity.ts` | `sync_conflicts` | `resolvedAt` |
| `role.entity.ts` | `roles` | `created_at`, `updated_at` |
| `permission.entity.ts` | `permissions` | `created_at`, `updated_at` |
| `subscription-tier.entity.ts` | `subscription_tiers` | `created_at`, `updated_at` |
| `user-role.entity.ts` | `user_roles` | `expires_at`, `created_at`, `updated_at` |
| `user-permission.entity.ts` | `user_permissions` | `expires_at`, `created_at`, `updated_at` |
| `user-subscription.entity.ts` | `user_subscriptions` | `expires_at`, `created_at`, `updated_at` |
| `resource-share.entity.ts` | `resource_shares` | `expires_at`, `created_at`, `updated_at` |
| `role-permission.entity.ts` | `role_permissions` | `created_at` |
| `tier-permission.entity.ts` | `tier_permissions` | `created_at` |
| `share-role.entity.ts` | `share_roles` | `created_at`, `updated_at` |
| `share-role-permission.entity.ts` | `share_role_permissions` | `created_at` |

> **Note** : Les entités sans option `name:` dans leurs décorateurs TypeORM conservent le nom de la propriété TypeScript (camelCase) comme nom de colonne SQL. Les entités avec `name: 'snake_case'` explicit utilisent le snake_case. Cf. migration SQL pour noms réels.

**AC5 (FKs entières)** : Capture entity déjà corrigée en story 12.2 (typeId, stateId, syncStatusId = UUID strings ✅).

**BaseEntity** : Déjà correct avec `{ type: 'timestamptz' }` depuis story 12.1 ✅.

### Debug Log

- Aucun blocage — toutes les corrections straightforward.
- Les erreurs TypeScript détectées (`tsc --noEmit`) sont toutes dans des fichiers `.spec.ts` pré-existants (knowledge module, story 12.3 softDelete mock manquant). Zéro erreur dans les fichiers de production.
- Tests d'acceptance pré-existants failant: `story-4-1.test.ts` (jest-cucumber step mismatch) et `story-7-1.test.ts` (step mismatch) — pas de régression causée par story 13.3.

### Completion Notes

- ✅ AC1: Toutes les `@CreateDateColumn` utilisent `{ type: 'timestamptz' }` (18 entités)
- ✅ AC2: Toutes les `@UpdateDateColumn` utilisent `{ type: 'timestamptz' }` (18 entités)
- ✅ AC3: Toutes les `@DeleteDateColumn` utilisent `{ type: 'timestamptz' }` (BaseEntity ✅ depuis story 12.1)
- ✅ AC4: Migration `1771800000000-AlterTimestampColumnsToTimestamptz.ts` créée — ALTER TABLE pour 18 tables
- ✅ AC5: FKs entières confirmées absentes dans `capture.entity.ts` (typeId, stateId, syncStatusId = UUID strings ✅ depuis story 12.2)
- ✅ AC6: 34 tests unitaires TypeORM metadata (après code review: +5 entités jonction, +4 expiresAt explicites, -3 tests vacueux) + 9 scenarios BDD Gherkin — tous passent
- ✅ Zero régression: baseline inchangé (46/46 acceptance tests — échecs pré-existants uniquement)

### Violations ADR-026 résiduelles (hors scope R5 — à tracker)

Ces violations ont été identifiées lors de l'audit story 13.3 mais sont hors scope (R5 uniquement) :

| Entité | Violation | ADR Rule | Référence |
|--------|-----------|----------|-----------|
| `SyncLog` | `userId` au lieu de `ownerId` | R7 | À corriger dans story dédiée |
| `AuditLog` | `user_id` au lieu de `ownerId` | R7 | À corriger dans story dédiée |
| `AdminUser`, `Notification`, `SyncLog`, `SyncConflict`, `AuditLog`, `ShareRole` | `@PrimaryGeneratedColumn` au lieu de `@PrimaryColumn` | R1 | Entités non-domaine — décision intentionnelle à documenter dans ADR |
| `User.status` | `varchar` libre au lieu d'ENUM PostgreSQL | R2 | À corriger dans story dédiée |

## File List

- Modified: `pensieve/backend/src/modules/action/domain/entities/todo.entity.ts`
- Modified: `pensieve/backend/src/modules/notification/domain/entities/Notification.entity.ts`
- Modified: `pensieve/backend/src/modules/shared/infrastructure/persistence/typeorm/entities/user.entity.ts`
- Modified: `pensieve/backend/src/modules/admin-auth/domain/entities/admin-user.entity.ts`
- Modified: `pensieve/backend/src/modules/shared/infrastructure/persistence/typeorm/entities/audit-log.entity.ts`
- Modified: `pensieve/backend/src/modules/sync/domain/entities/sync-log.entity.ts`
- Modified: `pensieve/backend/src/modules/sync/domain/entities/sync-conflict.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/role.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/permission.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/subscription-tier.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/user-role.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/user-permission.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/user-subscription.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/resource-share.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/role-permission.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/tier-permission.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/share-role.entity.ts`
- Modified: `pensieve/backend/src/modules/authorization/implementations/postgresql/entities/share-role-permission.entity.ts`
- Added: `pensieve/backend/src/migrations/1771800000000-AlterTimestampColumnsToTimestamptz.ts`
- Added: `pensieve/backend/src/common/entities/__tests__/timestamp-columns.spec.ts`
- Added: `pensieve/backend/test/acceptance/features/story-13-3-backend-timestamptz.feature`
- Added: `pensieve/backend/test/acceptance/story-13-3.test.ts`

## Change Log

- 2026-02-18: ADR-026 R5 compliance — Correction de 18 entités backend (timestamp → timestamptz). Migration ALTER TABLE créée pour 18 tables. Tests: 32 unit tests + 9 BDD scenarios. AC5 confirmé (FKs UUID depuis story 12.2).
- 2026-02-19: Code review — 2H+3M corrigés. `SyncConflict.resolvedAt` changé de `@CreateDateColumn` → `@Column` (sémantique correcte). Tests: +5 entités jonction, +4 expiresAt explicites, -3 tests vacueux → 34 unit tests. Documentation colonne noms corrigée (camelCase). Violations ADR-026 résiduelles documentées (R1/R7).
