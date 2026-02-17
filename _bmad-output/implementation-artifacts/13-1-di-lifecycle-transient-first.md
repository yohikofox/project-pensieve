# Story 13.1: Migrer le Container DI vers Transient First (ADR-021)

Status: backlog

## Story

As a **developer**,
I want **the TSyringe DI container to register services as Transient by default, with Singletons only for explicitly justified cases**,
So that **the architecture respects ADR-021 "Transient First" and avoids hidden shared state (ADR-021)**.

## Context

Audit ADR-021 (2026-02-17) révèle que l'intégralité du container (`container.ts`, 213 lignes) utilise `registerSingleton` pour tous les repositories et services applicatifs.

**Violations documentées** (`container.ts:130-155`) :
```typescript
container.registerSingleton(TOKENS.ICaptureRepository, CaptureRepository);
container.registerSingleton(TOKENS.IThoughtRepository, ThoughtRepository);
container.registerSingleton(RecordingService);
container.registerSingleton(TOKENS.IPermissionService, PermissionService);
// ... 15+ autres en registerSingleton
```

**Singletons légitimes (ne pas toucher)** :
- `eventBus` → `registerInstance` ✅
- `ExpoAudioAdapter` → `registerSingleton` ✅ (AudioRecorder = exception ADR)
- `SyncService` → `registerInstance` ✅ (état auth token justifié)
- `AudioUploadService` / `ChunkedUploadService` → factory pattern ✅

**Fichier cible** : `pensieve/mobile/src/infrastructure/di/container.ts`

## Acceptance Criteria

### AC1: Repositories migré en Transient
**Given** all repository registrations use `registerSingleton`
**When** I audit repositories for shared state
**Then** stateless repositories are changed to `container.register()` (Transient):
- `CaptureRepository`, `CaptureMetadataRepository`
- `ThoughtRepository`, `IdeaRepository`, `TodoRepository`
- Tous les repositories sans état partagé documenté

### AC2: Services applicatifs stateless migré en Transient
**Given** application services use `registerSingleton` without justification
**When** I audit services for shared state
**Then** stateless services are changed to `container.register()` (Transient):
- `PermissionService`, `FileStorageService`
- Services sans dépendances singleton exclusives
**And** services with legitimate state (e.g., `RecordingService` holding active recording) are documented and kept as Singleton with explicit comment

### AC3: Documentation inline des Singletons conservés
**Given** some services must remain as Singleton
**When** I keep them as `registerSingleton`
**Then** each remaining singleton has an inline comment explaining the justification:
```typescript
// SINGLETON: Maintains active recording state across the session (ADR-021 exception)
container.registerSingleton(RecordingService);
```

### AC4: Aucune régression de comportement
**Given** container registrations are changed from Singleton to Transient
**When** I run the full mobile test suite
**Then** all unit tests and BDD acceptance tests pass without modification
**And** the app starts correctly (no "container not registered" errors)

### AC5: Tests de non-régression DI
**Given** the container is updated
**When** I run specific DI resolution tests
**Then** lazy resolution inside hooks still works correctly
**And** Transient services receive fresh instances per resolution

## Tech Notes

- **Transient vs Singleton in TSyringe** :
  - `container.register(Token, { useClass: Impl })` = Transient (new instance per resolution)
  - `container.registerSingleton(Token, Impl)` = Singleton (same instance across app lifecycle)
- **Critère de décision** : Un service est Transient SAUF s'il (a) maintient un état partagé essentiel, (b) est explicitement listé comme exception dans ADR-021, ou (c) est coûteux à instancier (ex: WhisperService, AudioRecorder)
- **Tests** : Les tests d'acceptance utilisent des mocks — pas d'impact direct. Les tests unitaires peuvent être affectés si des singletons partagent de l'état entre tests → à vérifier
- **Lazy resolution** : Le pattern de résolution lazy dans les hooks (`container.resolve<T>(TOKEN)`) reste inchangé

## Related

- ADR-021: DI Lifecycle Strategy — Transient First
- `pensieve/mobile/src/infrastructure/di/container.ts`

## Definition of Done

- [ ] Tous les repositories stateless passés en `register()` (Transient)
- [ ] Services applicatifs stateless passés en Transient
- [ ] Singletons conservés documentés avec justification inline
- [ ] Tests mobiles (unit + acceptance) : zero régression
- [ ] App démarre correctement en simulateur
- [ ] ESLint + TypeScript strict passent
