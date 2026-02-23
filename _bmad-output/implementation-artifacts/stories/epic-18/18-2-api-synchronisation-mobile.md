# Story 18.2: API de synchronisation mobile (backend)

Status: backlog

## Story

As a **mobile app**,
I want **to send my device information to a backend endpoint and receive a targeted list of LLM model configurations and prompts**,
So that **the app always has up-to-date model configurations without requiring an app update**.

## Context

Le backend expose un endpoint `POST /api/llm-config/sync` qui :
1. Reçoit les informations matérielles du device (platform, NPU, RAM, etc.)
2. Applique les règles `LlmDeviceTargetingRule` pour filtrer les modèles pertinents
3. Retourne les modèles ciblés avec leurs prompts d'analyse associés

**Dépendance** : Story 18.1 (entités et migrations) doit être complétée avant.

**Auth** : JWT Better Auth requis (user authentifié). Les configs LLM sont par user (future extension possible).

## Acceptance Criteria

### AC1: Endpoint POST /api/llm-config/sync accessible
**Given** an authenticated user with a valid JWT
**When** `POST /api/llm-config/sync` is called with a valid `DeviceSyncRequestDto`
**Then** HTTP 200 is returned
**And** the response contains a list of `ModelConfigResponseDto`
**And** each config includes its associated `LlmAnalysisPrompt` entries

### AC2: Structure du DeviceSyncRequestDto validée
**Given** an authenticated request
**When** the body contains:
```json
{
  "platform": "android",
  "npuType": "tensor-tpu",
  "ramMb": 8192,
  "availableStorageMb": 16000,
  "appVersion": "1.2.3",
  "locale": "fr-FR",
  "deviceModelPattern": "Pixel 8"
}
```
**Then** all fields pass class-validator validation
**And** `platform` accepts only `ios | android`
**And** `npuType` accepts any string (open-ended, forward-compatible)
**And** numeric fields are integers >= 0

### AC3: Filtrage par règles de ciblage device
**Given** targeting rules exist in the database
**When** the sync is called with device info
**Then** only models matching the device's platform/npu/ram/pattern rules are returned
**And** models with `is_active = false` are excluded
**And** models are sorted: recommended first, then alphabetically by name

### AC4: Résolution des prompts d'analyse par modèle
**Given** a model with specific prompts AND global prompts exist
**When** the sync response is built
**Then** model-specific prompts (modelConfigId = model.id) take priority over global prompts (modelConfigId = null)
**And** all 4 analysis types (summary, highlights, action_items, ideas) are included per model
**And** only prompts with `is_active = true` are included

### AC5: Réponse structure ModelConfigResponseDto
**Given** a sync request
**When** models are returned
**Then** each model config includes all fields needed by the mobile app:
- `modelKey`, `name`, `description`, `filename`, `downloadUrl`
- `expectedSizeBytes`, `backend`, `promptTemplate`
- `deviceCompatibility`, `optimizedFor`
- `requiresAuth`, `category`, `isRecommended`
- `samplingParams` (temperature, topK, topP, maxTokens, randomSeed)
- `languages`, `specializations`, `strengths`, `weaknesses`
- `minRamMb`
- `analysisPrompts`: array of `{ analysisType, content, version }`

### AC6: Gestion des erreurs
**Given** an unauthenticated request
**When** `POST /api/llm-config/sync` is called
**Then** HTTP 401 is returned

**Given** an invalid request body
**When** validation fails
**Then** HTTP 400 is returned with validation details

**Given** no models match the device targeting rules
**When** the sync runs
**Then** HTTP 200 with an empty array is returned (le mobile utilise le fallback)

### AC7: Logging et observabilité
**Given** a sync request
**When** it is processed
**Then** device info (platform, npuType, appVersion) is logged at INFO level
**And** matched model count is logged
**And** Pino structured logger is used (ADR-observability)

## Tech Notes

- **Module** : `src/modules/llm-config/` (créé en 18.1)
- **Controller** : `LlmConfigController` à `src/modules/llm-config/infrastructure/http/`
- **Service** : `LlmConfigSyncService` — logique de filtrage et résolution des prompts
- **DTOs** : `DeviceSyncRequestDto`, `ModelConfigResponseDto`, `AnalysisPromptDto`
- **Auth Guard** : Utiliser `BetterAuthGuard` (Story 15.x)
- **Route** : `POST /api/llm-config/sync`
- **Pattern de ciblage** : Pour `device_model_pattern`, utiliser `ILIKE` ou regex PostgreSQL
- **Performance** : Eager loading des prompts dans la query (éviter N+1)

## Related

- Story 18.1: Entités et migrations (requis)
- Story 15.1: Better Auth server NestJS (auth guard)
- Story 18.4: Service de sync mobile (client de cet endpoint)
- ADR-026: Conformité data model
- ADR-observability: Logging Pino

## Definition of Done

- [ ] Endpoint `POST /api/llm-config/sync` implémenté
- [ ] `DeviceSyncRequestDto` avec validation class-validator complète
- [ ] `ModelConfigResponseDto` avec tous les champs nécessaires au mobile
- [ ] Logique de ciblage device fonctionnelle (filtrage par règles)
- [ ] Résolution prompts modèle-spécifique > global
- [ ] Auth JWT requis (401 si non authentifié)
- [ ] Tests BDD Gherkin (4 scénarios minimum)
- [ ] Tests unitaires pour `LlmConfigSyncService`
- [ ] Logging structuré Pino
