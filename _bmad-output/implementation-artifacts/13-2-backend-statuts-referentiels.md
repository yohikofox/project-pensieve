# Story 13.2: Créer les Tables Référentielles pour les Statuts Backend

Status: backlog

## Story

As a **developer**,
I want **entity statuses to use lookup tables with FK references, not free-text columns**,
So that **the data model is type-safe, extensible, and compliant with ADR-026 R2**.

## Context

Audit ADR-026 (2026-02-17) révèle :
- `thought.entity.ts:51` : `@Column({ type: 'text', name: '_status', default: 'active' })` — texte libre pour statut catégoriel
- `capture-state.entity.ts` : existe mais manque `label`, `display_order`, `is_active` (structure référentielle incomplète)

**Note** : Cette story est complémentaire à la Story 12.3 qui supprime le champ `_status` via soft delete. La story 12.3 gère la suppression des `deleted` via `deletedAt`. Cette story gère les statuts **fonctionnels** restants (ex : `active`, `processing`, `failed`...) en les migrant vers des tables référentielles.

## Acceptance Criteria

### AC1: Création de la table `thought_statuses`
**Given** `thought.entity.ts` uses free-text `_status`
**When** I create the reference table
**Then** a new table `thought_statuses` exists with columns:
- `id UUID PRIMARY KEY` (UUID domaine)
- `code VARCHAR(50) NOT NULL UNIQUE` (ex: 'active', 'archived')
- `label VARCHAR(100) NOT NULL` (ex: 'Actif', 'Archivé')
- `display_order INT NOT NULL DEFAULT 0`
- `is_active BOOLEAN NOT NULL DEFAULT true`

### AC2: FK sur `thought.entity.ts`
**Given** `thought_statuses` table is created
**When** I update `thought.entity.ts`
**Then** `thought.entity.ts` has a `status_id UUID FK → thought_statuses.id`
**And** the free-text `_status` column is removed (if not already done by story 12.3)

### AC3: Complétion de `capture_state` avec les champs manquants
**Given** `capture-state.entity.ts` exists but lacks `label`, `display_order`, `is_active`
**When** I update the entity and table
**Then** `capture_states` table gains:
- `label VARCHAR(100) NOT NULL`
- `display_order INT NOT NULL DEFAULT 0`
- `is_active BOOLEAN NOT NULL DEFAULT true`

### AC4: Seeding des données référentielles
**Given** the reference tables exist
**When** a seed script runs
**Then** `thought_statuses` is populated with at minimum: `active`, `archived`
**And** `capture_states` is populated with all existing states + their labels in French
**And** the seed is idempotent (can be run multiple times safely)

### AC5: Migration des données existantes
**Given** existing `thought` records have `_status = 'active'`
**When** the migration runs
**Then** existing records have their `status_id` set to the UUID of the 'active' thought_status record

### AC6: Tests BDD
**Given** reference tables are implemented
**When** I run acceptance tests
**Then** BDD scenarios validate:
- Creating a Thought assigns a valid `status_id`
- Invalid `status_id` is rejected at DB level (FK constraint)
- Status filtering works via JOIN on reference table

## Tech Notes

- **Pattern** : Tables référentielles avec seed immuable — ne jamais modifier les codes existants après mise en prod
- **Migration** : Deux migrations séparées : (1) création tables + seed, (2) migration données + ajout FK + suppression ancienne colonne
- **Seed** : Utiliser le mécanisme de seed existant du projet backend
- **Naming** : `thought_statuses` (pluriel, snake_case) — convention backend confirmée
- **FKs** : `ON DELETE RESTRICT` — un statut ne peut pas être supprimé s'il est utilisé

## Related

- Story 12.3: Soft Delete (supprime le cas 'deleted' via deletedAt — prérequis logique)
- ADR-026 R2: Tables référentielles pour les statuts catégoriels

## Definition of Done

- [ ] Table `thought_statuses` créée avec tous les champs
- [ ] Table `capture_states` complétée (`label`, `display_order`, `is_active`)
- [ ] `thought.entity.ts` utilise FK `status_id` vers `thought_statuses`
- [ ] Seed des données référentielles (idempotent)
- [ ] Migration des données existantes (`_status → status_id`)
- [ ] Tests BDD : min 3 scénarios
- [ ] Tests unitaires : FK constraint, seed idempotence
- [ ] Zero régression sur tests existants
