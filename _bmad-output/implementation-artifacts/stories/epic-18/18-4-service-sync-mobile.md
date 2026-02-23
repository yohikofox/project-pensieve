# Story 18.4: Service de synchronisation des configs LLM (mobile)

Status: backlog

## Story

As a **mobile user**,
I want **the app to automatically fetch up-to-date LLM model configurations from the backend at startup**,
So that **I always have the best model configurations for my device without needing to update the app**.

## Context

Le mobile implémente un service de synchronisation qui :
1. Collecte les informations matérielles du device (`DeviceInfoService`)
2. Appelle `POST /api/llm-config/sync` avec ces informations
3. Stocke le résultat dans des tables OP-SQLite dédiées
4. Utilise un fallback hardcodé minimal (SmolLM 135M) si la sync n'a jamais réussi

**Dépendance** : Story 18.2 (API backend) doit être complétée avant.

**Offline-first** : Si le device est hors ligne au démarrage, les configs en cache (OP-SQLite) sont utilisées. Si aucune sync n'a jamais réussi, le fallback minimal est utilisé.

## Acceptance Criteria

### AC1: DeviceInfoService collecte les informations du device
**Given** the app is running
**When** `DeviceInfoService.collect()` is called
**Then** it returns an object with:
- `platform`: `ios | android`
- `npuType`: détecté depuis le modèle device (ex: `tensor-tpu` pour Pixel, `apple-npu` pour iPhone)
- `ramMb`: RAM totale en MB (via `expo-device` ou API native)
- `availableStorageMb`: Stockage disponible
- `appVersion`: depuis `expo-constants`
- `locale`: depuis `expo-localization`
- `deviceModelPattern`: modèle device normalisé (ex: `Pixel 8`)
**And** `DeviceInfoService` is injectable via TSyringe (ADR-021 conforme)

### AC2: ModelConfigSyncService appelle le backend et stocke le résultat
**Given** the user is authenticated and the device is online
**When** `ModelConfigSyncService.sync()` is called
**Then** it calls `POST /api/llm-config/sync` with device info
**And** on success, it upserts the results into `llm_model_configs_remote` OP-SQLite table
**And** it upserts prompts into `llm_analysis_prompts_remote` OP-SQLite table
**And** it stores a `last_sync_at` timestamp in the local config store
**And** old records not returned by the server are marked as inactive (soft delete)

### AC3: Sync déclenchée au démarrage de l'app
**Given** the app starts
**When** the bootstrap sequence runs
**Then** `ModelConfigSyncService.sync()` is called asynchronously (non-bloquant)
**And** the UI does not wait for sync to complete before showing the home screen
**And** if the sync fails, an error is logged but no UI error is shown (silent failure)

### AC4: Fallback sur la config minimale embarquée
**Given** the sync has never succeeded (fresh install, no cache)
**When** the app tries to load model configs
**Then** a minimal hardcoded fallback config is returned:
  - 1 modèle: SmolLM 135M (ou modèle équivalent ultra-léger)
  - Prompts basiques pour les 4 types d'analyse
**And** this fallback is defined in a `MINIMAL_FALLBACK_CONFIG` constant

### AC5: Tables OP-SQLite créées via migration
**Given** a fresh installation
**When** the app starts for the first time
**Then** the following tables exist in OP-SQLite:
  - `llm_model_configs_remote`: même structure que l'entité backend + `is_local_active` boolean
  - `llm_analysis_prompts_remote`: model_key FK, analysis_type, content, version, is_active
**And** the migration runs via the existing OP-SQLite migration system

### AC6: Résolution de config après sync (priorité remote > fallback)
**Given** the sync has succeeded at least once
**When** the app needs to load model configs
**Then** `ModelConfigSyncService.getActiveConfigs()` returns remote configs from OP-SQLite
**And** if no remote configs are available (all inactive), the minimal fallback is returned
**And** this is transparent to the caller

### AC7: Interface IModelConfigSyncService et injection DI
**Given** the DI container setup
**When** `ModelConfigSyncService` is registered
**Then** it implements `IModelConfigSyncService` interface
**And** it is registered as `transient` (ADR-021 Transient First)
**And** it depends on `IDeviceInfoService`, `IHttpClient`, and `ILocalDatabase`

## Tech Notes

- **Services** :
  - `DeviceInfoService` : `src/contexts/knowledge/infrastructure/sync/DeviceInfoService.ts`
  - `ModelConfigSyncService` : `src/contexts/knowledge/infrastructure/sync/ModelConfigSyncService.ts`
  - Interfaces dans `src/contexts/knowledge/domain/sync/`
- **Tables OP-SQLite** :
  - `llm_model_configs_remote` + `llm_analysis_prompts_remote`
  - Migration dans le système existant (ex: `migrations/0010_llm_remote_configs.sql`)
- **DI Tokens** : `TOKENS.IDeviceInfoService`, `TOKENS.IModelConfigSyncService`
- **Bibliothèques** :
  - `expo-device` (RAM, modèle) — déjà dans le projet si disponible
  - `expo-constants` (app version) — déjà utilisé
  - `expo-localization` (locale) — déjà dans le projet
- **Fallback constant** : `src/contexts/knowledge/infrastructure/llmModelsConfig.ts` — ajouter `MINIMAL_FALLBACK_CONFIG`
- **HTTP** : Utiliser `fetchWithRetry` existant (ADR-025 conforme)

## Related

- Story 18.2: API backend (endpoint à appeler)
- Story 18.5: Intégration dans le pipeline LLM (consomme ce service)
- ADR-021: Transient First DI lifecycle
- ADR-025: fetchWithRetry HTTP wrapper
- `pensieve/mobile/src/infrastructure/http/fetchWithRetry.ts`
- `pensieve/mobile/src/infrastructure/di/container.ts`

## Definition of Done

- [ ] `DeviceInfoService` implémenté et injectable
- [ ] `ModelConfigSyncService` implémenté avec upsert OP-SQLite
- [ ] Sync déclenchée au démarrage (non-bloquante)
- [ ] Fallback minimal `MINIMAL_FALLBACK_CONFIG` défini
- [ ] Tables OP-SQLite créées via migration
- [ ] Interface `IModelConfigSyncService` et `IDeviceInfoService`
- [ ] Tests BDD Gherkin (4 scénarios : sync OK, sync offline, sync fail, fallback)
- [ ] Tests unitaires pour `DeviceInfoService` et `ModelConfigSyncService`
