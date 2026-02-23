# Story 18.3: Interface admin pour la gestion des configurations LLM

Status: backlog

## Story

As an **administrator**,
I want **a complete CRUD interface to manage LLM model configurations, analysis prompts, and device targeting rules**,
So that **I can update model parameters, prompts, and device targeting without deploying a new mobile app version**.

## Context

L'interface admin (Next.js dans `pensieve/admin/`) est étendue avec un nouveau module de gestion des configurations LLM. Les admins peuvent modifier les paramètres de sampling, améliorer les prompts d'analyse, ajouter de nouveaux modèles, et définir les règles de ciblage par device.

**Dépendance** : Story 18.1 (entités) et 18.2 (API backend) doivent être complétées, ou au moins 18.1.

**Parallélisation** : Cette story peut être développée en parallèle avec 18.2.

## Acceptance Criteria

### AC1: Liste des configurations de modèles LLM
**Given** I am logged in as an admin
**When** I navigate to `/admin/llm-config`
**Then** I see a table listing all `LlmModelConfig` entries
**And** the table shows: model key, name, backend, category, is_active, is_recommended
**And** I can filter by: is_active, backend, category
**And** I can search by model key or name

### AC2: Création d'une nouvelle configuration modèle
**Given** I am on the LLM config list page
**When** I click "Nouveau modèle"
**Then** a form opens with all fields from `LlmModelConfig`
**And** all required fields are validated before submission
**And** `samplingParams` is editable as a structured form (not raw JSON)
**And** `languages`, `specializations`, `strengths`, `weaknesses` are editable as tag inputs
**And** on success, I am redirected to the model detail page

### AC3: Modification d'une configuration existante
**Given** I am viewing a model config detail page
**When** I click "Modifier"
**Then** the edit form is pre-populated with current values
**And** I can update any field
**And** saving the form updates the record and shows a success toast

### AC4: Activation / désactivation d'un modèle
**Given** I am viewing a model in the list or detail page
**When** I toggle the "Actif" switch
**Then** the model's `is_active` flag is updated immediately
**And** inactive models are visually distinct in the list (grayed out)
**And** a confirmation is required before deactivating a recommended model

### AC5: Gestion des prompts d'analyse par modèle
**Given** I am on a model detail page
**When** I navigate to the "Prompts" tab
**Then** I see the 4 analysis types (summary, highlights, action_items, ideas)
**And** for each type, I see the active prompt content (model-specific if exists, else global)
**And** I can edit, create a new version, or reset to global for each prompt type
**And** creating a new version increments the `version` field and deactivates the previous

### AC6: Gestion des règles de ciblage device
**Given** I am on a model detail page
**When** I navigate to the "Ciblage device" tab
**Then** I see existing targeting rules for this model
**And** I can add a new rule (rule_type + rule_value)
**And** I can delete existing rules
**And** a preview section shows "Ce modèle sera proposé sur: Android (NPU tensor-tpu), iOS..."

### AC7: Aperçu des configs actives par type de device
**Given** I am on `/admin/llm-config/preview`
**When** I select a device profile (iOS / Android, RAM, NPU type)
**Then** I see the list of models that would be returned for that device
**And** the prompts that would be sent for each model are shown

## Tech Notes

- **App** : `pensieve/admin/` (Next.js)
- **Pages** :
  - `/admin/llm-config` — Liste des modèles
  - `/admin/llm-config/new` — Création
  - `/admin/llm-config/[id]` — Détail / édition
  - `/admin/llm-config/[id]/prompts` — Gestion prompts
  - `/admin/llm-config/[id]/targeting` — Règles ciblage
  - `/admin/llm-config/preview` — Prévisualisation
- **API calls** : Appeler le backend NestJS (REST). Créer des actions Server Actions Next.js ou API routes.
- **UI Components** : Utiliser les composants Shadcn/UI existants (Table, Form, Switch, Badge, Tabs)
- **Validation** : Zod pour les formulaires
- **Auth** : Better Auth côté admin (Story 15.3)

## Related

- Story 18.1: Entités backend (requis pour les types)
- Story 18.2: API backend (endpoints à appeler)
- Story 15.3: Web/Admin cleanup Supabase (auth admin)
- `pensieve/admin/` — codebase admin existant

## Definition of Done

- [ ] Page liste `/admin/llm-config` avec filtres et recherche
- [ ] Formulaire création/édition complet avec validation Zod
- [ ] Toggle activation/désactivation fonctionnel
- [ ] Gestion des prompts d'analyse (4 types, versioning)
- [ ] Gestion des règles de ciblage device (CRUD)
- [ ] Page de prévisualisation par profil device
- [ ] Tests E2E ou composant pour les flows principaux (3 scénarios minimum)
