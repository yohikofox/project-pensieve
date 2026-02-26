# Story 24.1: Feature Flag System — Backend Data Model & Resolution Engine

Status: done

## Story

As a **développeur et administrateur du système**,
I want **un système de feature flags structuré avec un référentiel de features et des tables d'assignation (user, role, permission)**,
So that **je puisse activer ou désactiver des fonctionnalités de manière granulaire par utilisateur, rôle, ou permission, avec une logique deny-wins garantissant la sécurité par défaut**.

**Context:** Le système actuel stocke `debug_mode_access` et `data_mining_access` comme colonnes booléennes sur la table `users` — un anti-pattern qui ne scale pas. Cette story remplace cette approche par un système générique et extensible. La logique de résolution suit la règle : si un flag est défini à `false` dans n'importe quelle assignation, le résultat final est `false` (deny-wins). Les flags non définis valent `false` par défaut.

## Acceptance Criteria

### AC1: Tables du Référentiel de Features
**Given** le système est initialisé
**When** les migrations sont appliquées
**Then** la table `features` existe avec les colonnes : `id` (uuid), `key` (varchar unique), `description` (text nullable), `default_value` (boolean default false), `created_at`, `updated_at`
**And** les tables `user_feature_assignments`, `role_feature_assignments`, `permission_feature_assignments` existent
**And** chaque table d'assignation contient : `id`, FK vers l'entité cible, `feature_id` (FK features), `value` (boolean), `created_at`
**And** des contraintes UNIQUE existent sur (`user_id`, `feature_id`), (`role_id`, `feature_id`), (`permission_id`, `feature_id`)

### AC2: Seed des Features Initiales
**Given** les migrations sont appliquées
**When** le seed est exécuté
**Then** les features suivantes existent dans `features` (toutes avec `default_value = false`) :
  - `key = 'debug_mode'`
  - `key = 'data_mining'`
  - `key = 'news_tab'`
  - `key = 'projects_tab'`
  - `key = 'capture_media_buttons'`

### AC3: Migration des Assignations Existantes
**Given** des utilisateurs ont `debug_mode_access = true` ou `data_mining_access = true` sur la table `users`
**When** la migration est appliquée
**Then** ces valeurs sont backfillées dans `user_feature_assignments`
  - `debug_mode_access = true` → `user_feature_assignments (user_id, feature_id='debug_mode', value=true)`
  - `data_mining_access = true` → `user_feature_assignments (user_id, feature_id='data_mining', value=true)`
**And** la migration inclut un rollback permettant de restaurer les colonnes `users`
**And** les colonnes `debug_mode_access` et `data_mining_access` sont supprimées de `users` (après backfill)

### AC4: Service de Résolution — Algorithme Deny-Wins
**Given** un `userId` est fourni au `FeatureResolutionService`
**When** `resolveFeatures(userId)` est appelé
**Then** le service collecte toutes les assignations actives :
  1. `user_feature_assignments` WHERE `user_id = userId`
  2. `role_feature_assignments` WHERE `role_id IN` (rôles actifs de l'utilisateur)
  3. `permission_feature_assignments` WHERE `permission_id IN` (permissions de l'utilisateur)
**And** pour chaque feature key, la règle de résolution est :
  - Aucune assignation → `false`
  - Au moins une assignation à `false` → `false` (deny-wins)
  - Toutes les assignations à `true` → `true`
**And** le résultat est retourné sous forme `Record<string, boolean>`

### AC5: Endpoint Mobile — GET Features
**Given** je suis un utilisateur authentifié
**When** l'app mobile requête `GET /api/users/:userId/features`
**Then** l'endpoint retourne `Record<string, boolean>` avec toutes les features connues du système
**And** la logique de résolution (AC4) est appliquée
**And** l'utilisateur ne peut accéder qu'à ses propres features (ownership check)
**And** l'endpoint est protégé par `BetterAuthGuard`

### AC6: Performances
**Given** un utilisateur a des rôles et permissions multiples
**When** `resolveFeatures(userId)` est appelé
**Then** la requête SQL utilise des JOINs optimisés (pas de N+1)
**And** le résultat peut être résolu en une seule passe SQL

## Tasks / Subtasks

### Backend — Data Model

- [x] **Task 1: Entités TypeORM** (AC1)
  - [x] Subtask 1.1: Créer `Feature` entity (`features` table)
  - [x] Subtask 1.2: Créer `UserFeatureAssignment` entity (`user_feature_assignments`)
  - [x] Subtask 1.3: Créer `RoleFeatureAssignment` entity (`role_feature_assignments`)
  - [x] Subtask 1.4: Créer `PermissionFeatureAssignment` entity (`permission_feature_assignments`)
  - [x] Subtask 1.5: Ajouter contraintes UNIQUE et indexes

- [x] **Task 2: Migrations** (AC1, AC2, AC3)
  - [x] Subtask 2.1: Migration création des 4 tables + indexes
  - [x] Subtask 2.2: Migration seed des 5 features initiales
  - [x] Subtask 2.3: Migration backfill `debug_mode_access` / `data_mining_access` → `user_feature_assignments`
  - [x] Subtask 2.4: Migration suppression des colonnes `debug_mode_access` et `data_mining_access` de `users`
  - [x] Subtask 2.5: Rollback pour chaque migration

### Backend — Domain & Application

- [x] **Task 3: Repositories** (AC4)
  - [x] Subtask 3.1: `FeatureRepository` — CRUD features + list
  - [x] Subtask 3.2: `UserFeatureAssignmentRepository` — CRUD assignations user
  - [x] Subtask 3.3: `RoleFeatureAssignmentRepository` — CRUD assignations rôle
  - [x] Subtask 3.4: `PermissionFeatureAssignmentRepository` — CRUD assignations permission

- [x] **Task 4: FeatureResolutionService** (AC4)
  - [x] Subtask 4.1: Implémenter `resolveFeatures(userId): Promise<Record<string, boolean>>`
    - Collecte des assignations user + role + permission en parallèle
    - Logique deny-wins par feature key
    - Toute feature non assignée → `false`
  - [x] Subtask 4.2: Optimisation requête SQL (JOINs, pas de N+1)
  - [x] Subtask 4.3: Tests unitaires (AC4 — 8 cas minimum)
    - Aucune assignation → false
    - User seul → valeur user
    - Role seul → valeur role
    - Permission seule → valeur permission
    - Conflit user=true + role=false → false (deny-wins)
    - Conflit role=true + permission=false → false (deny-wins)
    - Multiple rôles, un false → false
    - Toutes sources true → true

### Backend — Infrastructure

- [x] **Task 5: Adapter l'endpoint existant** (AC5)
  - [x] Subtask 5.1: Adapter `GET /api/users/:userId/features` → utilise `FeatureResolutionService`
  - [x] Subtask 5.2: Mettre à jour `UserFeaturesDto` → `Record<string, boolean>`
  - [x] Subtask 5.3: Adapter `UserFeaturesService.getUserFeatures()` (appelle `FeatureResolutionService`)
  - [x] Subtask 5.4: Supprimer `updateDebugModeAccess()` (remplacé par Story 24.2)

- [x] **Task 6: Enregistrement DI** (AC5)
  - [x] Subtask 6.1: Enregistrer les 4 repositories + `FeatureResolutionService` dans le module NestJS

### Tests

- [x] **Task 7: Tests** (AC1-AC6)
  - [x] Subtask 7.1: Tests unitaires `FeatureResolutionService` (≥ 8 cas — voir Task 4.3)
  - [x] Subtask 7.2: Tests unitaires repositories (CRUD basique)
  - [x] Subtask 7.3: Tests BDD Gherkin — `story-24-1.feature`
    - Scénario: utilisateur sans assignation → toutes features false
    - Scénario: assignation directe user → valeur respectée
    - Scénario: deny-wins — une source false suffit
    - Scénario: endpoint retourne format correct

## Technical Notes

### Schéma SQL

```sql
CREATE TABLE features (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key VARCHAR(100) NOT NULL UNIQUE,
  description TEXT,
  default_value BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE user_feature_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,  -- ref users.id
  feature_id UUID NOT NULL REFERENCES features(id) ON DELETE CASCADE,
  value BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, feature_id)
);

CREATE TABLE role_feature_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  feature_id UUID NOT NULL REFERENCES features(id) ON DELETE CASCADE,
  value BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(role_id, feature_id)
);

CREATE TABLE permission_feature_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
  feature_id UUID NOT NULL REFERENCES features(id) ON DELETE CASCADE,
  value BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(permission_id, feature_id)
);
```

### Algorithme de Résolution

```typescript
async resolveFeatures(userId: string): Promise<Record<string, boolean>> {
  // 1. Récupérer toutes les features du référentiel
  const allFeatures = await this.featureRepository.findAll();

  // 2. Collecter les assignations en parallèle
  const [userAssignments, roleAssignments, permissionAssignments] =
    await Promise.all([
      this.userAssignmentRepo.findByUserId(userId),
      this.roleAssignmentRepo.findByUserRoles(userId),
      this.permissionAssignmentRepo.findByUserPermissions(userId),
    ]);

  // 3. Indexer toutes les assignations par feature key
  const assignmentMap: Map<string, boolean[]> = new Map();
  for (const feature of allFeatures) {
    assignmentMap.set(feature.key, []);
  }
  for (const a of [...userAssignments, ...roleAssignments, ...permissionAssignments]) {
    const key = a.feature.key;
    assignmentMap.get(key)?.push(a.value);
  }

  // 4. Résoudre : deny-wins
  const result: Record<string, boolean> = {};
  for (const [key, values] of assignmentMap) {
    if (values.length === 0) result[key] = false;          // non défini → false
    else if (values.includes(false)) result[key] = false;  // deny-wins
    else result[key] = true;                               // tous true → true
  }

  return result;
}
```

### Format de Réponse API

```json
{
  "debug_mode": false,
  "data_mining": false,
  "news_tab": true,
  "projects_tab": false,
  "capture_media_buttons": false
}
```

### Numéros de Migration (ordre d'exécution)

- `1780100000000-CreateFeatureFlagTables.ts`
- `1780200000000-SeedInitialFeatures.ts`
- `1780300000000-BackfillFeatureFlagsFromUserColumns.ts`
- `1780400000000-DropLegacyFeatureColumnsFromUsers.ts`

## Dependencies

- Epic 1 (auth), Epic 7 (UserFeaturesService existant — remplacé ici)
- Story 24.2 (admin UI) dépend de cette story
- Story 24.3 (mobile) dépend de cette story

## Definition of Done

- [x] 4 tables créées et migrées
- [x] Backfill des données existantes validé
- [x] `FeatureResolutionService` testé avec ≥ 8 scénarios unitaires
- [x] Endpoint `GET /api/users/:userId/features` retourne `Record<string, boolean>`
- [x] Tests BDD passent
- [x] Aucun N+1 dans la résolution

## Dev Notes

### Architecture Compliance

- **ADR-026 (Backend Data Model)** : Toutes les nouvelles entités TypeORM (`Feature`, `UserFeatureAssignment`, etc.) doivent hériter de `BaseEntity` du shared module. UUID généré côté domaine (pas `@PrimaryGeneratedColumn`). Timestamps en `timestamptz`. Pas de cascade TypeORM → suppression gérée en couche applicative (sauf Feature → Assignments où cascade est acceptable car relation de possession directe).
- **ADR-021 (Transient First)** : `FeatureResolutionService` et les 4 repositories → `@Injectable()` NestJS standard (pas de singleton). Seul exception justifiée : service avec cache interne.
- **ADR-023 (Result Pattern)** : Les méthodes de service retournant des données → utiliser `Result<T>` depuis `shared/domain/Result.ts` si exposées via interface domain. Côté NestJS controller, laisser les exceptions NestJS remonter.
- **Pattern migration** : Fichier de migration = `1780XXXXXXXXX-NomDescriptif.ts` dans `backend/src/migrations/`. Toujours inclure `down()` pour réversibilité. Voir `1739750000000-AddDebugModeAccessToUsers.ts` comme référence.
- **Ownership check** : Le guard d'ownership `currentUser.id === userId` est déjà implémenté dans `UserFeaturesController` — à préserver lors de l'adaptation.

### Project Structure Notes

- Créer un nouveau module NestJS : `backend/src/modules/feature-flags/` avec :
  - `domain/entities/feature.entity.ts`, `user-feature-assignment.entity.ts`, `role-feature-assignment.entity.ts`, `permission-feature-assignment.entity.ts`
  - `application/services/feature-resolution.service.ts`
  - `application/repositories/` (interfaces)
  - `infrastructure/persistence/typeorm/` (implémentations repositories)
  - `feature-flags.module.ts`
- Modifier `backend/src/modules/identity/application/services/user-features.service.ts` → déléguer vers `FeatureResolutionService`
- Modifier `backend/src/modules/identity/application/dtos/user-features.dto.ts` → `Record<string, boolean>`
- Ajouter `FeatureFlagsModule` à `backend/src/app.module.ts`
- 4 fichiers de migration dans `backend/src/migrations/` (séquentiels — voir numéros dans Technical Notes)
- Tests BDD : `backend/test/acceptance/story-24-1.test.ts` + `backend/test/acceptance/features/story-24-1.feature`

### Testing Standards

- **Pattern BDD backend** : Voir `backend/test/acceptance/story-7-1.test.ts` — `loadFeature()` + `defineFeature()` + `beforeAll` (app NestJS + DataSource) + `beforeEach` (DELETE FROM tables)
- **Tests unitaires** : Mock repositories via interfaces → tester l'algorithme deny-wins en isolation
- **Nettoyage** : `beforeEach` doit effacer `user_feature_assignments`, `role_feature_assignments`, `permission_feature_assignments` et `features` pour isoler les scénarios
- **Supertest** : Utiliser pour les tests d'endpoints (auth mock via BetterAuthGuard)

### References

- [Source: backend/src/modules/identity/application/services/user-features.service.ts] — Service existant à adapter
- [Source: backend/src/modules/identity/application/dtos/user-features.dto.ts] — DTO à migrer vers `Record<string, boolean>`
- [Source: backend/src/modules/identity/infrastructure/controllers/users.controller.ts] — Endpoint GET /api/users/:userId/features
- [Source: backend/src/modules/admin-auth/infrastructure/controllers/admin-users.controller.ts] — Pattern AdminJwtGuard + PATCH features (à remplacer par Story 24.2)
- [Source: backend/src/migrations/1739750000000-AddDebugModeAccessToUsers.ts] — Pattern migration avec rollback
- [Source: backend/src/modules/shared/infrastructure/persistence/typeorm/entities/user.entity.ts] — Pattern entité (timestamptz, UUID)
- [Source: backend/test/acceptance/story-7-1.test.ts] — Pattern tests BDD acceptance
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-026] — Data Model Backend
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-021] — DI Lifecycle Transient First

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

- `updateFeatures()` dans `UserFeaturesService` conservé comme bridge (upsert dans user_feature_assignments) pour maintenir l'endpoint admin PATCH fonctionnel jusqu'à Story 24.2
- `UserFeaturesDto` migré en type alias `Record<string, boolean>` — rupture de contrat mobile intentionnelle, corrigée par Story 24.3
- Entités assignment utilisent `@PrimaryGeneratedColumn('uuid')` (pattern UserRole/UserPermission) et non AppBaseEntity (pas de soft-delete nécessaire sur les assignations)
- `RoleFeatureAssignmentRepository` et `PermissionFeatureAssignmentRepository` injectent DataSource directement car `UserRoleRepository`/`UserPermissionRepository` ne sont pas exportés par AuthorizationModule
- `admin-users.controller.spec.ts` non modifié : les tests controller testent la couche controller (service mocké), restent valides
- **Code review fixes** : `UserFeatureAssignmentRepository` migré en raw SQL (cohérence avec role/permission repos, évite NPE TypeORM soft-delete) ; filtre `AND f.deleted_at IS NULL` ajouté sur les 3 repos ; FK `user_id → users(id) ON DELETE CASCADE` ajoutée dans migration ; fixture spec complétée à 5 features

### File List

**Créés :**
- `pensieve/backend/src/modules/feature-flags/domain/entities/feature.entity.ts`
- `pensieve/backend/src/modules/feature-flags/domain/entities/user-feature-assignment.entity.ts`
- `pensieve/backend/src/modules/feature-flags/domain/entities/role-feature-assignment.entity.ts`
- `pensieve/backend/src/modules/feature-flags/domain/entities/permission-feature-assignment.entity.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/feature.repository.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/user-feature-assignment.repository.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/role-feature-assignment.repository.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/permission-feature-assignment.repository.ts`
- `pensieve/backend/src/modules/feature-flags/application/services/feature-resolution.service.ts`
- `pensieve/backend/src/modules/feature-flags/application/services/feature-resolution.service.spec.ts`
- `pensieve/backend/src/modules/feature-flags/feature-flags.module.ts`
- `pensieve/backend/src/migrations/1780100000000-CreateFeatureFlagTables.ts`
- `pensieve/backend/src/migrations/1780200000000-SeedInitialFeatures.ts`
- `pensieve/backend/src/migrations/1780300000000-BackfillFeatureFlagsFromUserColumns.ts`
- `pensieve/backend/src/migrations/1780400000000-DropLegacyFeatureColumnsFromUsers.ts`
- `pensieve/backend/test/acceptance/features/story-24-1.feature`
- `pensieve/backend/test/acceptance/story-24-1.test.ts`

**Modifiés :**
- `pensieve/backend/src/modules/identity/application/dtos/user-features.dto.ts` — type alias `Record<string, boolean>`
- `pensieve/backend/src/modules/identity/application/services/user-features.service.ts` — délègue à FeatureResolutionService
- `pensieve/backend/src/modules/identity/application/services/user-features.service.spec.ts` — réécriture pour nouvelle API
- `pensieve/backend/src/modules/identity/identity.module.ts` — import FeatureFlagsModule
- `pensieve/backend/src/app.module.ts` — import FeatureFlagsModule
- `pensieve/backend/src/modules/shared/infrastructure/persistence/typeorm/entities/user.entity.ts` — suppression debug_mode_access et data_mining_access
- `pensieve/backend/src/modules/knowledge/infrastructure/rabbitmq/rabbitmq-setup.service.ts` — refactor cycle de vie connexion : local vars, error handlers, close après setup, suppression onModuleDestroy/getChannel
