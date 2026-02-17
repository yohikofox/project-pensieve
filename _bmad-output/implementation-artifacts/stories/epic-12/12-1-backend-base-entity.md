# Story 12.1: Créer la BaseEntity Partagée Backend

Status: ready-for-dev

## Story

As a **developer**,
I want **a shared BaseEntity that all backend entities inherit from**,
So that **`id`, `createdAt`, `updatedAt`, `deletedAt` are defined once and consistently across all entities (ADR-026 R6)**.

## Context

Audit ADR-026 (2026-02-17) révèle l'absence totale de `BaseEntity` dans le backend. Les champs `id`, `createdAt`, `updatedAt` sont redéclarés dans chaque entité (`capture.entity.ts`, `thought.entity.ts`, `idea.entity.ts`, `todo.entity.ts`...).

Cette story est le **prérequis bloquant** pour les stories 12.2 et 12.3 : une BaseEntity cohérente simplifie la migration des PKs et du soft delete.

**Fichier cible** : `pensieve/backend/src/common/entities/base.entity.ts`

## Acceptance Criteria

### AC1: Création du fichier BaseEntity
**Given** the backend has no shared BaseEntity
**When** I implement `src/common/entities/base.entity.ts`
**Then** it contains:
- `id: string` — `@PrimaryColumn('uuid')` (UUID généré dans le domaine, pas par la DB)
- `createdAt: Date` — `@CreateDateColumn({ type: 'timestamptz' })`
- `updatedAt: Date` — `@UpdateDateColumn({ type: 'timestamptz' })`
- `deletedAt: Date | null` — `@DeleteDateColumn({ type: 'timestamptz' })`
- Classe abstraite annotée avec `@Entity()` parent non résolu (ou mixin)

### AC2: Migration TypeORM cohérente
**Given** the BaseEntity is defined
**When** a new TypeORM migration is generated
**Then** all tables that inherit BaseEntity have:
- `id UUID NOT NULL PRIMARY KEY` (sans `DEFAULT gen_random_uuid()` — l'UUID vient du domaine)
- `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
- `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
- `deleted_at TIMESTAMPTZ NULL`

### AC3: Héritage dans au moins une entité de référence
**Given** the BaseEntity is created
**When** I apply it to `capture.entity.ts` as pilot entity
**Then** the entity no longer redeclares `id`, `createdAt`, `updatedAt`, `deletedAt`
**And** all existing unit tests still pass without modification

### AC4: Tests unitaires de la BaseEntity
**Given** the BaseEntity is implemented
**When** I run unit tests
**Then** there are tests validating that:
- Inheriting entities have the correct column types
- `deletedAt` is nullable by default
- No duplicate column declarations are present

## Tech Notes

- **Pattern** : `abstract class BaseEntity { ... }` avec décorateurs TypeORM
- **PK** : `@PrimaryColumn('uuid')` — l'UUID doit être fourni par la couche domaine (factory pattern ou valeur initiale dans le constructeur)
- **Soft delete** : TypeORM `@DeleteDateColumn` + `withDeleted()` / `softDelete()` sur les repositories
- **Migration** : Ne PAS modifier les migrations existantes. Créer une nouvelle migration pour chaque entité pilote de cette story (capture seulement)
- **Scope de cette story** : Créer la BaseEntity + appliquer sur `capture.entity.ts` uniquement (les autres entités sont couvertes par stories 12.2 et 12.3)

## Related

- Story 12.2: Remplacer `@PrimaryGeneratedColumn` par `@PrimaryColumn` + UUID domaine
- Story 12.3: Implémenter soft delete sur toutes les entités
- ADR-026: Backend Data Model Design Rules

## Definition of Done

- [ ] `src/common/entities/base.entity.ts` créé
- [ ] `capture.entity.ts` hérite de `BaseEntity` (entité pilote)
- [ ] Migration TypeORM créée et validée (non appliquée en prod)
- [ ] Tests unitaires (min 3 cas)
- [ ] Tests BDD si un AC utilisateur est impacté
- [ ] ESLint + TypeScript strict passent
- [ ] Aucune régression sur les tests existants
