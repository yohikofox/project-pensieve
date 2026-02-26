# Story 24.2: Feature Flag System — Admin API & Interface d'Administration

Status: done

## Story

As a **administrateur du système**,
I want **une interface d'administration permettant de gérer le catalogue de features et d'assigner des features à des utilisateurs, rôles ou permissions**,
So that **je puisse activer ou désactiver des fonctionnalités pour des utilisateurs spécifiques (ex: mon frère) sans modifier le code ni redéployer l'application**.

**Context:** Story 24.1 crée le modèle de données. Cette story expose les endpoints admin et adapte l'interface admin existante pour remplacer les anciens toggles booléens (`debug_mode_access`, `data_mining_access`) par le nouveau système, et ajouter la gestion des 3 nouvelles features (`news_tab`, `projects_tab`, `capture_media_buttons`).

## Acceptance Criteria

### AC1: API Admin — Catalogue de Features
**Given** je suis connecté en tant qu'admin
**When** je requête `GET /api/admin/features`
**Then** je reçois la liste complète des features avec leur `key`, `description`, `default_value`
**And** `POST /api/admin/features` permet de créer une nouvelle feature (key unique, description, default_value)
**And** `PATCH /api/admin/features/:id` permet de modifier description et default_value (key immuable)

### AC2: API Admin — Assignations par Utilisateur
**Given** je suis connecté en tant qu'admin
**When** je consulte les features d'un utilisateur via `GET /api/admin/users/:userId/features`
**Then** je reçois la liste des features avec :
  - `value` : valeur de résolution finale (deny-wins)
  - `sources` : trace de résolution (quelles assignations ont contribué)
  - `assignments` : assignations directes sur cet utilisateur uniquement
**And** `PUT /api/admin/users/:userId/features/:featureKey` crée ou met à jour une assignation directe (value: boolean)
**And** `DELETE /api/admin/users/:userId/features/:featureKey` supprime une assignation directe (retour au résultat via rôle/permission)

### AC3: API Admin — Assignations par Rôle
**Given** je suis connecté en tant qu'admin
**When** je requête `GET /api/admin/roles/:roleId/features`
**Then** je reçois les assignations features du rôle
**And** `PUT /api/admin/roles/:roleId/features/:featureKey` crée ou met à jour l'assignation
**And** `DELETE /api/admin/roles/:roleId/features/:featureKey` supprime l'assignation

### AC4: API Admin — Assignations par Permission
**Given** je suis connecté en tant qu'admin
**When** je requête `GET /api/admin/permissions/:permissionId/features`
**Then** je reçois les assignations features de la permission
**And** `PUT /api/admin/permissions/:permissionId/features/:featureKey` crée ou met à jour l'assignation
**And** `DELETE /api/admin/permissions/:permissionId/features/:featureKey` supprime l'assignation

### AC5: Admin UI — Page Catalogue de Features
**Given** je suis connecté à l'interface admin
**When** j'accède à la section "Feature Flags"
**Then** je vois la liste de toutes les features avec key, description, default_value
**And** je peux créer une nouvelle feature via un formulaire
**And** je peux modifier la description d'une feature existante

### AC6: Admin UI — Gestion des Features par Utilisateur
**Given** je suis sur la page détail d'un utilisateur dans l'admin
**When** je consulte la section "Feature Flags"
**Then** les anciens toggles `debug_mode_access` et `data_mining_access` sont remplacés par la liste complète des features
**And** chaque feature affiche : nom, valeur effective (résolution), source(s) contributrices
**And** je peux activer/désactiver une assignation directe sur cet utilisateur (toggle)
**And** si aucune assignation directe, le toggle est en état "non défini" (distinct de false)
**And** un indicateur visuel distingue : assigné directement / hérité de rôle / hérité de permission / non défini

### AC7: Sécurité
**Given** n'importe quelle requête sur `/api/admin/features/**` ou `/api/admin/*/features`
**When** le requêteur n'est pas un admin authentifié
**Then** la requête retourne 401 (non authentifié) ou 403 (non autorisé)
**And** tous les endpoints sont protégés par `AdminJwtGuard`

## Tasks / Subtasks

### Backend — Admin Endpoints

- [x] **Task 1: AdminFeaturesController** (AC1)
  - [x] Subtask 1.1: `GET /api/admin/features` — liste catalogue
  - [x] Subtask 1.2: `POST /api/admin/features` — créer feature
  - [x] Subtask 1.3: `PATCH /api/admin/features/:id` — modifier feature
  - [x] Subtask 1.4: DTOs + validation class-validator

- [x] **Task 2: Admin User Features Endpoints** (AC2)
  - [x] Subtask 2.1: `GET /api/admin/users/:userId/features` — features effectives + trace
  - [x] Subtask 2.2: `PUT /api/admin/users/:userId/features/:featureKey` — upsert assignation directe
  - [x] Subtask 2.3: `DELETE /api/admin/users/:userId/features/:featureKey` — supprimer assignation directe
  - [x] Subtask 2.4: Adapter le `PATCH /api/admin/users/:userId/features` existant (compatibilité ou suppression)

- [x] **Task 3: Admin Role Features Endpoints** (AC3)
  - [x] Subtask 3.1: `GET /api/admin/roles/:roleId/features`
  - [x] Subtask 3.2: `PUT /api/admin/roles/:roleId/features/:featureKey`
  - [x] Subtask 3.3: `DELETE /api/admin/roles/:roleId/features/:featureKey`

- [x] **Task 4: Admin Permission Features Endpoints** (AC4)
  - [x] Subtask 4.1: `GET /api/admin/permissions/:permissionId/features`
  - [x] Subtask 4.2: `PUT /api/admin/permissions/:permissionId/features/:featureKey`
  - [x] Subtask 4.3: `DELETE /api/admin/permissions/:permissionId/features/:featureKey`

- [x] **Task 5: Tests Backend** (AC1-AC4, AC7)
  - [x] Subtask 5.1: Tests unitaires controllers (mocks services)
  - [x] Subtask 5.2: Tests BDD `story-24-2.feature`
    - Scénario: créer feature + assigner à user + vérifier résolution
    - Scénario: non-admin → 403
    - Scénario: supprimer assignation → retour à false

### Admin UI

- [x] **Task 6: Page Catalogue Feature Flags** (AC5)
  - [x] Subtask 6.1: Route `/admin/features`
  - [x] Subtask 6.2: Liste features avec key / description / default_value
  - [x] Subtask 6.3: Formulaire création feature
  - [x] Subtask 6.4: Inline edit description

- [x] **Task 7: Section Feature Flags sur la page utilisateur** (AC6)
  - [x] Subtask 7.1: Remplacer les anciens toggles `debug_mode_access` / `data_mining_access`
  - [x] Subtask 7.2: Composant `FeatureFlagRow` : key + valeur effective + source + toggle assignation directe
  - [x] Subtask 7.3: État "non défini" distinct de false (ex: badge gris vs rouge)
  - [x] Subtask 7.4: Indicateur source (direct / rôle / permission / non défini)
  - [x] Subtask 7.5: Requêtes API avec feedback (loading, success, erreur)

- [x] **Task 8: Sections Feature Flags sur pages rôle et permission** (AC3, AC4)
  - [x] Subtask 8.1: Section "Feature Flags" sur la page détail d'un rôle
  - [x] Subtask 8.2: Section "Feature Flags" sur la page détail d'une permission
  - [x] Subtask 8.3: Composant réutilisable `FeatureFlagAssignments`

## Technical Notes

### Trace de Résolution (Response Format)

```json
GET /api/admin/users/:userId/features
{
  "news_tab": {
    "resolved": true,
    "sources": [
      { "type": "user", "value": true },
      { "type": "role", "roleId": "...", "roleName": "beta", "value": true }
    ]
  },
  "debug_mode": {
    "resolved": false,
    "sources": []
  }
}
```

### Upsert Assignation Directe

```json
PUT /api/admin/users/:userId/features/news_tab
Body: { "value": true }
→ 200 OK : { "key": "news_tab", "value": true, "source": "user" }
```

### Suppression Assignation

```json
DELETE /api/admin/users/:userId/features/news_tab
→ 204 No Content (l'assignation directe est supprimée, la résolution via rôle/permission prend le relai)
```

## Dependencies

- Story 24.1 (data model + FeatureResolutionService) doit être `done`
- Admin UI existante (framework utilisé dans `pensieve/admin/`)

## Definition of Done

- [x] Tous les endpoints admin implémentés et sécurisés (AdminJwtGuard)
- [x] Tests unitaires controllers passent
- [x] Scénarios BDD passent
- [x] Admin UI affiche et permet de gérer les feature flags par utilisateur
- [x] Anciens toggles `debug_mode_access` / `data_mining_access` supprimés de l'UI admin
- [x] Composant de trace de résolution visible dans l'UI

## Dev Notes

### Architecture Compliance

- **AdminJwtGuard** : Tous les endpoints `admin/features/**` et `admin/*/features` doivent être annotés `@UseGuards(AdminJwtGuard)`. Voir pattern dans `admin-auth/infrastructure/controllers/admin-users.controller.ts`.
- **DTOs** : Utiliser `class-validator` (`@IsString()`, `@IsBoolean()`, `@IsOptional()`) sur tous les DTOs. Pas de getters — champs publics directs (pattern observé dans le projet).
- **Upsert pattern** : `PUT` endpoint → comportement upsert (créer ou mettre à jour). Retourner 200 avec l'assignation, jamais 201 (car pas de location header nécessaire ici).
- **`DELETE` → 204 No Content** : Suppression d'une assignation directe → 204 sans body.
- **ADR-021** : `AdminFeaturesController` → `@Injectable()` standard NestJS. `FeatureResolutionService` injecté (vient de Story 24.1).
- **Compatibilité** : L'ancien `PATCH /api/admin/users/:userId/features` (UpdateUserFeaturesDto avec `debugModeAccess`, `dataMiningAccess`) doit être **supprimé** ou remplacé — pas de coexistence.

### Project Structure Notes

- **Backend** : Ajouter les routes admin dans le module `feature-flags` créé en Story 24.1 :
  - `feature-flags/infrastructure/controllers/admin-features.controller.ts` — catalogue + CRUD
  - `feature-flags/infrastructure/controllers/admin-user-features.controller.ts` — assignations user
  - `feature-flags/infrastructure/controllers/admin-role-features.controller.ts` — assignations rôle
  - `feature-flags/infrastructure/controllers/admin-permission-features.controller.ts` — assignations permission
- Adapter `admin-auth/infrastructure/controllers/admin-users.controller.ts` → supprimer `PATCH :userId/features`
- **Admin UI** : Framework à identifier dans `pensieve/admin/src/` (Next.js 15, App Router). Créer :
  - `admin/src/app/features/page.tsx` — catalogue feature flags
  - Composant `FeatureFlagRow` réutilisable avec toggle + indicateur source
  - Composant `FeatureFlagAssignments` pour pages rôle/permission
- Tests BDD backend : `backend/test/acceptance/story-24-2.test.ts` + `.feature`
- Tests unitaires : `backend/src/modules/feature-flags/infrastructure/controllers/__tests__/`

### Testing Standards

- **Unit tests controllers** : Mocker `FeatureResolutionService` et les repositories. Tester 401/403 en simulant absence/présence du guard.
- **BDD Scénarios minimum** :
  1. Admin crée feature + assigne à user + vérifie résolution correcte
  2. Non-admin accède → 403 Forbidden
  3. Suppression assignation directe → résolution retombe à `false`
- **Admin UI** : Tests React Testing Library si possible pour les composants de toggle

### References

- [Source: backend/src/modules/admin-auth/infrastructure/controllers/admin-users.controller.ts] — Pattern AdminJwtGuard + endpoints admin existants
- [Source: backend/src/modules/feature-flags/] — Module créé en Story 24.1 (prérequis)
- [Source: backend/test/acceptance/story-7-1.test.ts] — Pattern BDD backend acceptance
- [Source: pensieve/admin/src/] — Structure Admin UI Next.js (à explorer au démarrage)
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-026] — Data Model Backend
- [Source: _bmad-output/implementation-artifacts/stories/epic-24/24-1-feature-flag-system-backend.md] — Story prérequis avec schéma SQL et algorithme deny-wins

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

**Code Review — Fixes appliqués (2026-02-26)**

- **C1 (CRITIQUE)** — `getUserFeatures` retournait `Record<string, boolean>` au lieu de `{ resolved, sources }`. Corrigé dans `admin-feature-flags.service.ts` + `admin-user-features.controller.ts`. La requête SQL ajoute maintenant `'user'/'role'/'permission' AS "sourceType"`. Tests unitaires mis à jour.
- **C2 (CRITIQUE)** — Algorithm deny-wins ignorait `defaultValue`. Corrigé : fallback sur `f.defaultValue` quand `sources.length === 0`.
- **H1 (HAUT)** — AC6 partiellement non implémenté (pas de trace source, pas de bouton delete). Corrigé dans `users/page.tsx` : affichage `resolved`, badges source (direct/rôle/permission/non défini), bouton ✕ pour supprimer assignation directe.
- **H2 (HAUT)** — Zéro scénario BDD pour AC3 (rôle) et AC4 (permission). Ajouté Scénarios 4 et 5 dans `story-24-2.feature` + step definitions dans `story-24-2.test.ts`.
- **H3 (HAUT)** — Bouton inline edit toujours invisible (missing `group` class). Corrigé dans `features/page.tsx` ligne 143.
- **M2 (MOYEN)** — `@IsNotEmpty()` sur boolean supprimé dans `upsert-feature-assignment.dto.ts`.
- **M1 (MOYEN — non corrigé)** — Double `@Controller('api/admin/users')` entre `AdminUserFeaturesController` et `AdminUsersController` : anti-pattern NestJS. Refactoring scope > cette story.

### File List

**Backend — Nouveaux fichiers**
- `pensieve/backend/src/modules/feature-flags/application/dtos/create-feature.dto.ts`
- `pensieve/backend/src/modules/feature-flags/application/dtos/update-feature.dto.ts`
- `pensieve/backend/src/modules/feature-flags/application/dtos/upsert-feature-assignment.dto.ts`
- `pensieve/backend/src/modules/feature-flags/application/services/admin-feature-flags.service.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/admin-features.controller.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/admin-user-features.controller.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/admin-role-features.controller.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/admin-permission-features.controller.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/__tests__/admin-features.controller.spec.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/__tests__/admin-user-features.controller.spec.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/__tests__/admin-role-features.controller.spec.ts`
- `pensieve/backend/src/modules/feature-flags/infrastructure/controllers/__tests__/admin-permission-features.controller.spec.ts`
- `pensieve/backend/test/acceptance/features/story-24-2.feature`
- `pensieve/backend/test/acceptance/story-24-2.test.ts`

**Backend — Fichiers modifiés**
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/feature.repository.ts` (ajout findById/create/update)
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/user-feature-assignment.repository.ts` (ajout upsert/deleteAssignment)
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/role-feature-assignment.repository.ts` (ajout findByRoleId/upsert/deleteAssignment)
- `pensieve/backend/src/modules/feature-flags/infrastructure/persistence/typeorm/permission-feature-assignment.repository.ts` (ajout findByPermissionId/upsert/deleteAssignment)
- `pensieve/backend/src/modules/feature-flags/feature-flags.module.ts` (export AdminFeatureFlagsService)
- `pensieve/backend/src/modules/admin-auth/admin-auth.module.ts` (import FeatureFlagsModule + enregistrement 4 controllers)
- `pensieve/backend/src/modules/admin-auth/infrastructure/controllers/admin-users.controller.ts` (suppression getUserFeatures/updateUserFeatures legacy)
- `pensieve/backend/src/modules/admin-auth/infrastructure/controllers/admin-users.controller.spec.ts` (suppression tests legacy)

**Backend — Fichiers supprimés**
- `pensieve/backend/src/modules/admin-auth/application/dtos/update-user-features.dto.ts`

**Admin UI — Nouveaux fichiers**
- `pensieve/admin/app/(dashboard)/features/page.tsx`
- `pensieve/admin/components/admin/feature-flag-assignments.tsx`

**Admin UI — Fichiers modifiés**
- `pensieve/admin/lib/api-client.ts` (nouveaux endpoints + types + 204 handling + UserFeatureResolution type)
- `pensieve/admin/app/(dashboard)/users/page.tsx` (trace UI avec sources + bouton delete assignation directe)
- `pensieve/admin/app/(dashboard)/roles/page.tsx` (bouton Feature Flags + Dialog)
- `pensieve/admin/app/(dashboard)/permissions/page.tsx` (bouton Feature Flags + Dialog)
- `pensieve/admin/components/admin/sidebar-nav.tsx` (ajout lien Feature Flags)
- `pensieve/admin/app/(dashboard)/features/page.tsx` (fix group-hover inline edit)
