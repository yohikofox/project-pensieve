# Story 18.5: Intégration des configs remote dans le pipeline LLM (mobile)

Status: backlog

## Story

As a **user**,
I want **the app to use the remotely-synchronized LLM configurations for transcription analysis**,
So that **improvements to AI prompts and model parameters are applied automatically without a new app release**.

## Context

Le pipeline LLM existant (`PostProcessingService`, `CaptureAnalysisService`) utilise actuellement les configs hardcodées depuis `llmModelsConfig.ts`. Cette story relie les configs synchronisées (Story 18.4) au pipeline existant :
- `PostProcessingService` lit les configs modèles depuis OP-SQLite (remote) en priorité
- `CaptureAnalysisService` utilise les prompts remote en priorité, fallback sur les prompts locaux
- `SamplingParams` et `ModelPromptProfile` sont appliqués depuis les configs remote

**Dépendance** : Story 18.4 (service de sync) doit être complétée avant.

**Backward compatibility** : Le fallback minimal embarqué garantit 100% de fonctionnalité même sans sync réussie.

## Acceptance Criteria

### AC1: PostProcessingService utilise les configs remote
**Given** remote model configs are available in OP-SQLite (sync succeeded)
**When** `PostProcessingService` selects a model for inference
**Then** it reads model configs from `IModelConfigSyncService.getActiveConfigs()`
**And** the selected model's `samplingParams` (temperature, topK, topP, maxTokens) are applied to the inference call
**And** the remote config takes priority over the hardcoded `llmModelsConfig.ts`

### AC2: CaptureAnalysisService utilise les prompts remote
**Given** remote analysis prompts are available in OP-SQLite for the selected model
**When** `CaptureAnalysisService` builds the prompt for a given analysis type
**Then** it uses the remote prompt for that model + analysis type
**And** if no model-specific prompt exists, it falls back to the global remote prompt
**And** `{{CURRENT_DATE}}` is substituted with the current date at runtime

### AC3: Fallback transparent sur configs locales
**Given** no remote sync has ever succeeded (fresh install)
**When** `PostProcessingService` or `CaptureAnalysisService` needs configs
**Then** `MINIMAL_FALLBACK_CONFIG` from Story 18.4 is used seamlessly
**And** no error or degraded UI state is shown to the user
**And** the analysis pipeline runs normally with the fallback config

### AC4: SamplingParams appliqués dynamiquement
**Given** a model config with `samplingParams = { temperature: 0.7, topK: 40, maxTokens: 512 }`
**When** an LLM inference call is made
**Then** those exact parameters are passed to the LLM backend (llamarn, mediapipe, or litert-lm)
**And** the `samplingParams` are not hardcoded anywhere in the pipeline code
**And** each LLM backend adapter correctly maps the unified `SamplingParams` to its specific API

### AC5: ModelPromptProfile construit depuis les configs remote
**Given** a model with remote config and prompts
**When** building the prompt for a capture analysis
**Then** a `ModelPromptProfile` is constructed with:
- `promptTemplate` (chatml, gemma, phi, llama)
- `systemPrompt` from remote `LlmAnalysisPrompt.content`
- `samplingParams` from remote `LlmModelConfig.samplingParams`
**And** this profile is passed to the LLM backend adapter

### AC6: Tests acceptance — chemin remote
**Given** remote configs are available in the test context
**When** the analysis pipeline runs
**Then** the remote prompt is used in the LLM call
**And** the remote sampling params are applied
**And** BDD Gherkin scenario: `remote_config_used_in_pipeline.feature`

### AC7: Tests acceptance — chemin fallback
**Given** no remote configs are available (mock returns empty)
**When** the analysis pipeline runs
**Then** the minimal fallback config is used
**And** the pipeline completes without error
**And** BDD Gherkin scenario: `fallback_config_used_in_pipeline.feature`

## Tech Notes

- **Services à modifier** :
  - `PostProcessingService` : Injecter `IModelConfigSyncService`, remplacer accès à `llmModelsConfig.ts`
  - `CaptureAnalysisService` : Injecter `IModelConfigSyncService`, utiliser `getPromptsForModel(modelKey, analysisType)`
- **Nouveaux types** :
  - `SamplingParams`: `{ temperature: number; topK?: number; topP?: number; maxTokens: number; randomSeed?: number }`
  - `ModelPromptProfile`: `{ promptTemplate: string; systemPrompt: string; samplingParams: SamplingParams }`
- **Méthode à ajouter dans IModelConfigSyncService** :
  - `getActiveConfigs(): Promise<ModelConfigDto[]>`
  - `getPromptsForModel(modelKey: string, analysisType: AnalysisType): Promise<string>`
- **Substitution `{{CURRENT_DATE}}`** : Dans `CaptureAnalysisService` avant appel LLM
- **Adapters LLM** : Vérifier que llamarn, mediapipe et litert-lm acceptent les `samplingParams` dynamiques

## Related

- Story 18.4: Service de sync mobile (requis — fournit IModelConfigSyncService)
- Story 13.4: Result Pattern (conformité pour les retours de service)
- `pensieve/mobile/src/contexts/knowledge/application/PostProcessingService.ts`
- `pensieve/mobile/src/contexts/knowledge/application/CaptureAnalysisService.ts`
- `pensieve/mobile/src/contexts/knowledge/infrastructure/llmModelsConfig.ts`

## Definition of Done

- [ ] `PostProcessingService` refactorisé pour utiliser `IModelConfigSyncService`
- [ ] `CaptureAnalysisService` refactorisé pour utiliser les prompts remote
- [ ] Types `SamplingParams` et `ModelPromptProfile` définis dans le domaine
- [ ] `{{CURRENT_DATE}}` substitué dynamiquement dans les prompts
- [ ] Adapters LLM (llamarn, mediapipe, litert-lm) supportent `SamplingParams` dynamiques
- [ ] Tests BDD — chemin remote (2 scénarios minimum)
- [ ] Tests BDD — chemin fallback (2 scénarios minimum)
- [ ] Code review adversariale (3-10 issues trouvées, corrigées)
- [ ] `llmModelsConfig.ts` n'est plus utilisé directement dans le pipeline (uniquement pour le seed initial ou la migration)
