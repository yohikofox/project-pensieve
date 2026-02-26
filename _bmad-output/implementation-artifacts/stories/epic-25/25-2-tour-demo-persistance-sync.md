# Story 25.2: Tour Démo — Persistance Locale & Synchronisation Backend

Status: ready-for-dev

## Story

En tant qu'**utilisateur de l'application**,
je veux que **mon statut de tour démo soit persisté localement et synchronisé avec le backend**,
afin de **ne plus revoir le tour après l'avoir complété, même après un wipe local ou un changement de device**.

Dépendance : **Story 25.1 doit être complète** (TourService, TourCompleted/TourSkipped events).

## Acceptance Criteria

### AC1 : Persistance locale immédiate (offline-first)
**Given** l'utilisateur vient de compléter ou passer le tour
**When** l'événement `TourCompleted` ou `TourSkipped` est émis sur l'EventBus
**Then** l'état est écrit dans OP-SQLite immédiatement (table `tour_preferences`)
**And** l'écriture est synchrone (ADR-018 OP-SQLite synchronous)
**And** les champs stockés sont : `tourCompleted: boolean`, `tourVersion: string`, `tourCompletedAt: Date`

### AC2 : Lecture au démarrage (guard local)
**Given** l'application démarre
**When** `TourProvider` (story 25.1) interroge l'état du tour
**Then** il lit d'abord `TourPreferenceRepository` local
**And** si `tourCompleted === true` ET `tourVersion === CURRENT_TOUR_VERSION`, le tour ne se déclenche pas

### AC3 : Synchronisation backend (best-effort, async)
**Given** l'utilisateur vient de compléter ou passer le tour ET est connecté
**When** la persistance locale est terminée
**Then** un appel `PATCH /api/users/:userId/tour` est envoyé en arrière-plan
**And** l'échec réseau N'EMPÊCHE PAS la persistance locale
**And** le corps de la requête contient : `{ completed: true, version: "1.0", completedAt: ISO8601 }`

### AC4 : Récupération depuis le backend (fallback wipe)
**Given** les données locales ont été effacées (wipe OP-SQLite)
**When** l'application démarre et vérifie l'état du tour
**Then** si la table locale est vide, un appel `GET /api/users/:userId/tour` est effectué
**And** si le backend répond `tourCompleted: true`, le tour n'est pas déclenché
**And** la réponse est mise en cache localement pour les prochains démarrages

### AC5 : Backend — Colonnes User entity
**Given** la migration TypeORM est appliquée
**When** la table `users` est inspectée
**Then** 3 nouvelles colonnes existent : `tour_completed BOOLEAN DEFAULT false`, `tour_version VARCHAR(20) NULLABLE`, `tour_completed_at TIMESTAMPTZ NULLABLE`

### AC6 : Backend — Endpoint PATCH /api/users/:userId/tour
**Given** un utilisateur authentifié effectue un PATCH
**When** le corps contient `{ completed: true, version: "1.0" }`
**Then** les colonnes `tour_completed`, `tour_version`, `tour_completed_at` sont mises à jour
**And** la réponse HTTP 200 retourne l'état mis à jour
**And** un utilisateur ne peut modifier que SES propres données (ownership check)

### AC7 : Backend — Endpoint GET /api/users/:userId/tour
**Given** un utilisateur authentifié effectue un GET
**When** la requête est valide
**Then** la réponse retourne : `{ tourCompleted: boolean, tourVersion: string | null, tourCompletedAt: string | null }`
**And** ownership check : un utilisateur ne peut lire que ses propres données

### AC8 : Mode offline — pas de retry bloquant
**Given** l'utilisateur complète le tour sans connexion réseau
**When** la synchro backend échoue
**Then** l'état local est préservé
**And** aucune interface de retry n'est affichée à l'utilisateur
**And** la prochaine synchro tentée sera au prochain démarrage connecté

## Tasks / Subtasks

### Mobile — Couche Domaine

- [ ] **Task 1 : TourPreference domain model** (AC1, AC2)
  - [ ] Créer `src/contexts/identity/tour/domain/TourPreference.ts`
  - [ ] Champs : `tourCompleted: boolean`, `tourVersion: string`, `tourCompletedAt: Date | null`
  - [ ] Créer `src/contexts/identity/tour/domain/ITourPreferenceRepository.ts` (interface)
  - [ ] Méthodes : `save(pref: TourPreference): void`, `find(): TourPreference | null`
  - [ ] Ajouter token `TOKENS.ITourPreferenceRepository` dans `tokens.ts`

### Mobile — Repository OP-SQLite

- [ ] **Task 2 : TourPreferenceRepository** (AC1, AC2)
  - [ ] Créer `src/contexts/identity/tour/data/TourPreferenceRepository.ts`
  - [ ] Table : `tour_preferences` (id INTEGER PK, tour_completed INTEGER, tour_version TEXT, tour_completed_at TEXT)
  - [ ] Méthode `save()` : upsert (INSERT OR REPLACE) synchrone via OP-SQLite
  - [ ] Méthode `find()` : SELECT, retourne null si vide
  - [ ] Ajouter la migration dans le système de migrations existant
  - [ ] Enregistrer comme `transient` dans `container.ts` (repository = transient, ADR-021)

### Mobile — Service de Synchronisation

- [ ] **Task 3 : TourSyncService** (AC3, AC4, AC8)
  - [ ] Créer `src/contexts/identity/tour/services/TourSyncService.ts`
  - [ ] Méthode `syncToBackend(pref: TourPreference, userId: string, token: string): Promise<void>`
    - [ ] Utilise `fetchWithRetry` (ADR-025 — fetch natif uniquement)
    - [ ] Endpoint : `PATCH ${EXPO_PUBLIC_API_URL}/api/users/${userId}/tour`
    - [ ] Silencieux en cas d'erreur (log warning, pas d'exception propagée)
  - [ ] Méthode `fetchFromBackend(userId: string, token: string): Promise<TourPreference | null>`
    - [ ] Endpoint : `GET ${EXPO_PUBLIC_API_URL}/api/users/${userId}/tour`
    - [ ] Retourne null si erreur réseau ou 404
  - [ ] Enregistrer comme `transient` dans `container.ts`

### Mobile — Intégration TourService (Story 25.1)

- [ ] **Task 4 : Brancher persistance dans TourService** (AC1, AC2, AC4)
  - [ ] Dans `TourService`, injecter `ITourPreferenceRepository` et `TourSyncService`
  - [ ] Dans le handler EventBus `TourCompleted` / `TourSkipped` :
    - [ ] Écrire localement via `repository.save()`
    - [ ] Déclencher `syncService.syncToBackend()` en arrière-plan (fire-and-forget)
  - [ ] Dans `shouldShowTour()` :
    - [ ] Lire local d'abord via `repository.find()`
    - [ ] Si null ET connecté : appeler `syncService.fetchFromBackend()` puis mettre en cache local
    - [ ] Appliquer la logique : `if (forceFromSettings) return true` sinon vérifier local/version

### Backend — Entity

- [ ] **Task 5 : Colonnes User entity** (AC5)
  - [ ] Modifier `src/modules/shared/infrastructure/persistence/typeorm/entities/user.entity.ts`
  - [ ] Ajouter 3 colonnes :
    ```typescript
    @Column({ type: 'boolean', default: false })
    tourCompleted!: boolean;

    @Column({ type: 'varchar', length: 20, nullable: true })
    tourVersion?: string | null;

    @Column({ type: 'timestamptz', nullable: true })
    tourCompletedAt?: Date | null;
    ```

### Backend — Migration

- [ ] **Task 6 : Migration TypeORM** (AC5)
  - [ ] Créer migration dans `src/migrations/`
  - [ ] Nom : `AddTourColumnsToUsers`
  - [ ] `ALTER TABLE users ADD COLUMN tour_completed BOOLEAN DEFAULT false NOT NULL`
  - [ ] `ALTER TABLE users ADD COLUMN tour_version VARCHAR(20)`
  - [ ] `ALTER TABLE users ADD COLUMN tour_completed_at TIMESTAMPTZ`
  - [ ] Migration `down()` : DROP COLUMN (rollback propre)

### Backend — DTOs

- [ ] **Task 7 : DTOs tour** (AC6, AC7)
  - [ ] Créer `src/modules/identity/application/dtos/tour-preference.dto.ts`
  - [ ] `UpdateTourDto` : `{ completed: boolean, version: string }` (avec class-validator)
  - [ ] `TourPreferenceResponseDto` : `{ tourCompleted: boolean, tourVersion: string | null, tourCompletedAt: string | null }`

### Backend — Service

- [ ] **Task 8 : TourPreferenceService** (AC6, AC7)
  - [ ] Créer `src/modules/identity/application/services/tour-preference.service.ts`
  - [ ] Injecter `InjectRepository(User)` → `Repository<User>`
  - [ ] Méthode `updateTour(userId: string, dto: UpdateTourDto): Promise<TourPreferenceResponseDto>`
  - [ ] Méthode `getTour(userId: string): Promise<TourPreferenceResponseDto>`

### Backend — Controller

- [ ] **Task 9 : Endpoints dans UsersController** (AC6, AC7)
  - [ ] Modifier `src/modules/identity/infrastructure/controllers/users.controller.ts`
  - [ ] Ajouter `PATCH /api/users/:userId/tour` → `TourPreferenceService.updateTour()`
  - [ ] Ajouter `GET /api/users/:userId/tour` → `TourPreferenceService.getTour()`
  - [ ] Les deux endpoints : `@UseGuards(BetterAuthGuard)` + ownership check (`currentUser.id !== userId → ForbiddenException`)

### Backend — Module

- [ ] **Task 10 : Enregistrer dans IdentityModule** (AC6, AC7)
  - [ ] Ajouter `TourPreferenceService` aux providers dans `identity.module.ts`
  - [ ] Ajouter `TypeOrmModule.forFeature([User])` si pas déjà présent

### Tests Mobile

- [ ] **Task 11 : Unit tests TourPreferenceRepository** (AC1, AC2)
  - [ ] `src/contexts/identity/tour/data/TourPreferenceRepository.test.ts`
  - [ ] Mock OP-SQLite, couvrir : save, find (présent / absent)

- [ ] **Task 12 : Unit tests TourSyncService** (AC3, AC4, AC8)
  - [ ] Mock `fetchWithRetry`, couvrir : succès, erreur réseau silencieuse, fetch fallback

- [ ] **Task 13 : BDD/Gherkin** (AC1, AC2, AC3, AC4, AC8)
  - [ ] `tests/acceptance/features/story-25-2.feature`
  - [ ] `tests/acceptance/story-25-2.test.ts`
  - [ ] Scenarios : persistance après complétion, guard local, fallback backend wipe, offline silencieux

### Tests Backend

- [ ] **Task 14 : Unit tests TourPreferenceService** (AC6, AC7)
  - [ ] `src/modules/identity/application/services/tour-preference.service.spec.ts`
  - [ ] Mock `Repository<User>`, couvrir updateTour et getTour

- [ ] **Task 15 : BDD/Gherkin Backend** (AC6, AC7)
  - [ ] `test/acceptance/features/story-25-2.feature`
  - [ ] `test/acceptance/story-25-2.test.ts`
  - [ ] Scenarios : PATCH tour, GET tour, ownership violation (403)

## Dev Notes

### Pattern Repository OP-SQLite (ADR-018)

```typescript
// Synchronous queries — pas d'async/await
save(pref: TourPreference): void {
  const db = container.resolve<IDatabase>(TOKENS.IDatabase);
  db.execute(
    `INSERT OR REPLACE INTO tour_preferences
     (id, tour_completed, tour_version, tour_completed_at)
     VALUES (1, ?, ?, ?)`,
    [pref.tourCompleted ? 1 : 0, pref.tourVersion, pref.tourCompletedAt?.toISOString() ?? null]
  );
}
```

### Logique shouldShowTour (dans TourService)

```typescript
async shouldShowTour(forceFromSettings: boolean): Promise<boolean> {
  if (forceFromSettings) return true;

  // 1. Check local
  const local = this.repository.find();
  if (local?.tourCompleted && local.tourVersion === CURRENT_TOUR_VERSION) return false;

  // 2. Si local vide, check backend (fallback wipe)
  if (!local) {
    const remote = await this.syncService.fetchFromBackend(userId, token);
    if (remote?.tourCompleted && remote.tourVersion === CURRENT_TOUR_VERSION) {
      this.repository.save(remote); // Cache local
      return false;
    }
  }

  return true;
}
```

### Backend — Ownership pattern existant

```typescript
// Pattern identique à l'endpoint /features existant
if (currentUser.id !== userId) {
  throw new ForbiddenException('You can only access your own tour preferences');
}
```

### Project Structure Notes

- Nouveaux fichiers dans `src/contexts/identity/tour/` (DDD: Identity BC)
- Backend : modifications `user.entity.ts` + `users.controller.ts` + nouvelle migration
- Ne pas modifier `src/contexts/shared/tour/` de story 25.1 (couche UI séparée)

### References

- [Source: architecture.md — ADR-018] OP-SQLite synchronous queries, offline-first
- [Source: architecture.md — ADR-025] fetchWithRetry pour appels HTTP
- [Source: users.controller.ts] Pattern ownership check existant (ligne 38-42)
- [Source: user.entity.ts] Colonnes notification preferences (pattern suivi)
- [Source: pensieve/backend/CLAUDE.md] TypeORM synchronize:false, migrations obligatoires

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List
