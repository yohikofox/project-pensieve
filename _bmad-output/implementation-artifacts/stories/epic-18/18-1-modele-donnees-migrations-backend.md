# Story 18.1: Modèle de données et migrations backend

Status: backlog

## Story

As a **backend developer**,
I want **TypeORM entities and migrations for LLM model configurations, analysis prompts, and device targeting rules**,
So that **administrators can manage model configurations through the database without app updates**.

## Context

Les paramètres de sampling, les system prompts d'analyse et les métadonnées de modèles sont actuellement en dur dans `llmModelsConfig.ts`. Cette story extrait ces données vers le backend en créant un modèle de données TypeORM propre, conforme à l'ADR-026.

**Point d'entrée du changement** : Cette story est le fondement de l'Epic 18. Les stories 18.2 (API), 18.3 (Admin UI) et 18.4 (Mobile sync) en dépendent directement.

**Seed initial** : Reprendre l'intégralité de `MODEL_CONFIGS` et `ANALYSIS_PROMPTS` actuels depuis `llmModelsConfig.ts` mobile pour pré-peupler la base.

## Acceptance Criteria

### AC1: Entité LlmModelConfig créée et conforme ADR-026
**Given** the backend codebase
**When** the migration runs
**Then** a table `llm_model_configs` exists with all required columns:
- `id` UUID (uuidv7, PK, généré dans le domaine)
- `model_key` string UNIQUE (ex: `gemma3-1b-mediapipe`)
- `name`, `description`, `filename`, `download_url` strings
- `expected_size_bytes` bigint
- `backend` enum: `llamarn | mediapipe | litert-lm`
- `prompt_template` enum: `chatml | gemma | phi | llama`
- `device_compatibility` enum: `all | google | apple`
- `optimized_for` enum: `apple | google` NULLABLE
- `requires_auth` boolean
- `category` enum: `qwen | llama | gemma | other`
- `is_active` boolean DEFAULT true
- `is_recommended` boolean DEFAULT false
- `sampling_params` jsonb (`{ temperature, topK?, topP?, maxTokens, randomSeed? }`)
- `languages` text[]
- `specializations`, `strengths`, `weaknesses` text[]
- `min_ram_mb` integer NULLABLE
- `created_at`, `updated_at` timestamptz (hérités de BaseEntity ADR-026)

### AC2: Entité LlmAnalysisPrompt créée et conforme ADR-026
**Given** the backend codebase
**When** the migration runs
**Then** a table `llm_analysis_prompts` exists with:
- `id` UUID (uuidv7, PK)
- `model_config_id` UUID NULLABLE FK → `llm_model_configs` (null = prompt global par défaut)
- `analysis_type` enum: `summary | highlights | action_items | ideas`
- `content` text (prompt complet, supporte `{{CURRENT_DATE}}`)
- `version` integer DEFAULT 1
- `is_active` boolean DEFAULT true
- `created_at`, `updated_at` timestamptz
**And** une contrainte UNIQUE sur `(model_config_id, analysis_type, version)`

### AC3: Entité LlmDeviceTargetingRule créée et conforme ADR-026
**Given** the backend codebase
**When** the migration runs
**Then** a table `llm_device_targeting_rules` exists with:
- `id` UUID (uuidv7, PK)
- `model_config_id` UUID FK → `llm_model_configs` (NOT NULL)
- `rule_type` enum: `npu_type | platform | min_ram_mb | device_model_pattern`
- `rule_value` string (ex: `tensor-tpu`, `android`, `6144`, `Pixel [67]`)
- `created_at`, `updated_at` timestamptz

### AC4: Migration TypeORM UP et DOWN fonctionnelles
**Given** a clean database
**When** `npm run migration:run` is executed
**Then** all 3 tables are created with correct constraints and indexes
**And** `npm run migration:revert` restores the previous state

### AC5: Seed initial reprenant les configs existantes
**Given** the migration has run
**When** `npm run seed` is executed
**Then** all existing model configs from `llmModelsConfig.ts` are inserted in `llm_model_configs`
**And** all existing analysis prompts are inserted in `llm_analysis_prompts` (avec `model_config_id = null` pour les prompts globaux)
**And** device targeting rules basées sur les champs `deviceCompatibility` et `optimizedFor` existants sont insérées

### AC6: Conformité ADR-026
**Given** the new entities
**When** reviewed against ADR-026
**Then** all entities extend `BaseEntity` (created_at, updated_at, deleted_at)
**And** PKs are UUIDs generated in the domain layer (uuidv7)
**And** no TypeORM cascade is used
**And** no `@PrimaryGeneratedColumn` is used

## Tech Notes

- **Module NestJS** : Créer `src/modules/llm-config/` (nouveau bounded context)
- **Entités** : `src/modules/llm-config/domain/entities/`
- **Migration** : Nommage convention `YYYYMMDDHHMMSS-add-llm-config-tables.ts`
- **Seed** : TypeORM seeder ou script ts-node depuis `database/seeds/`
- **BaseEntity** : Utiliser la `BaseEntity` créée en Story 12.1
- **Indexes recommandés** : `model_key` (unique), `is_active`, `(model_config_id, analysis_type)` sur prompts

## Related

- Story 12.1: BaseEntity partagée backend (requis)
- Story 12.2: UUID uuidv7 (pattern requis)
- Story 18.2: API de synchronisation (dépend de cette story)
- Story 18.3: Admin UI (dépend de cette story)
- `pensieve/mobile/src/contexts/knowledge/infrastructure/llmModelsConfig.ts` — source du seed

## Definition of Done

- [ ] 3 entités TypeORM créées et conformes ADR-026
- [ ] Migration UP et DOWN fonctionnelles (`npm run migration:run` + `migration:revert`)
- [ ] Seed initial avec toutes les configs existantes
- [ ] Tests unitaires pour les entités (validation schéma, contraintes)
- [ ] Tests BDD (3 scénarios minimum — ex: insertion modèle, prompt global, règle targeting)
- [ ] Module `llm-config` enregistré dans `AppModule`
