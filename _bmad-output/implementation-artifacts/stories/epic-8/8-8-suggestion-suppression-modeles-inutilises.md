# Story 8.8 : Suggestion de Suppression des Modèles Inutilisés

Status: ready-for-dev

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

En tant qu'**utilisateur ayant téléchargé plusieurs modèles LLM et/ou Whisper**,
je veux **être alerté visuellement quand un modèle n'a pas été utilisé depuis plus de 15 jours**,
afin de **libérer de l'espace disque en supprimant les modèles devenus inutiles**.

**GitHub Issue :** [#8 - feat: Suggestion de suppression des modèles inutilisés (15 jours)](https://github.com/yohikofox/pensieve/issues/8)

## Contexte

### Problème

Les modèles LLM (ex: Qwen2.5 3B = ~2Go) et Whisper (ex: large-v3 = ~3.1Go) occupent un espace disque considérable. L'utilisateur peut télécharger plusieurs modèles et ne plus en utiliser certains, sans s'en rendre compte. Il n'existe actuellement aucun mécanisme pour suggérer la suppression des modèles inutilisés.

### Ce qui est déjà en place

#### Tracking existant
**⚠️ AUCUN tracking `lastUsed` n'existe actuellement.** Les services stockent uniquement :

```typescript
// LLMModelService.ts — AsyncStorage keys existantes
"@pensieve/llm_model_postprocessing"     // modèle sélectionné pour post-processing
"@pensieve/llm_model_analysis"           // modèle sélectionné pour analyse
"@pensieve/download_resume_${modelId}"   // état de récupération de téléchargement

// TranscriptionModelService.ts — AsyncStorage keys existantes
"@pensieve/selected_whisper_model"       // modèle Whisper sélectionné
```

#### Services de gestion des modèles
- **`LLMModelService.ts`** (833 lignes) — download, delete, select, pause/resume, recovery. Méthodes clés : `downloadModel()`, `deleteModel()`, `setModelForTask()`, `isModelDownloaded()`, `getModelPath()`
- **`TranscriptionModelService.ts`** (503 lignes) — idem pour Whisper. Méthodes clés : `downloadModel()`, `deleteModel()`, `setSelectedModel()`, `isModelDownloaded()`

#### UI existante
- **`LLMSettingsScreen.tsx`** — liste des modèles LLM avec Download/Pause/Resume/Cancel/Delete
- **`WhisperSettingsScreen.tsx`** — liste des modèles Whisper (idem)
- **`LLMModelCard.tsx`** — composant carte réutilisable avec progress bar
- **`WhisperModelCard.tsx`** — composant carte Whisper

#### Stores existants
- **`settingsStore.ts`** (Zustand) — préférences globales, modèles LLM/Whisper persistés AsyncStorage
- **`llmSettingsScreenStore.ts`** (Zustand) — état UI-spécifique LLM screen

#### Infrastructure DI
- **`container.ts`** — TSyringe, enregistrements existants des services Normalization
- **`tokens.ts`** — tokens Symbol pour injection

### Comportement attendu après Story 8.8

```
Utilisateur :
  1. Télécharge SmolLM 135M et Qwen2.5 3B
  2. Utilise quotidiennement SmolLM 135M
  3. 15 jours passent sans utiliser Qwen2.5 3B
  4. ✅ Carte Qwen2.5 3B affiche un badge orange "Non utilisé depuis 15 jours"
  5. ✅ Message explicatif : "Ce modèle n'a pas été utilisé depuis 15 jours"
  6. ✅ Bouton "Supprimer (2.0 Go)" et bouton "Ignorer"
  7. Si "Supprimer" → fichier supprimé, espace libéré, UI mise à jour
  8. Si "Ignorer" → alerte masquée, ne réapparaît pas sauf si modèle réutilisé
```

## Acceptance Criteria

### AC1 : Tracking de la date de dernière utilisation — LLM

**Given** un modèle LLM est téléchargé pour la première fois
**When** le téléchargement se termine avec succès
**Then** la date de dernière utilisation est initialisée à la date du jour (date de téléchargement = première disponibilité)
**And** la clé `@pensieve/model_last_used_llm_${modelId}` est créée en AsyncStorage avec le timestamp ISO

**Given** l'utilisateur sélectionne un modèle LLM dans `LLMSettingsScreen` (`setModelForTask()`)
**When** la sélection est persistée
**Then** la clé `@pensieve/model_last_used_llm_${modelId}` est mise à jour avec le timestamp actuel
**And** le badge d'inactivité sur ce modèle est réinitialisé (il ne peut plus être "inutilisé")

### AC2 : Tracking de la date de dernière utilisation — Whisper

**Given** un modèle Whisper est téléchargé pour la première fois
**When** le téléchargement se termine avec succès
**Then** la date de dernière utilisation est initialisée à la date du jour
**And** la clé `@pensieve/model_last_used_whisper_${modelSize}` est créée en AsyncStorage

**Given** l'utilisateur sélectionne un modèle Whisper (`setSelectedModel()`)
**When** la sélection est persistée
**Then** la clé `@pensieve/model_last_used_whisper_${modelSize}` est mise à jour

### AC3 : Détection des modèles inutilisés après 15 jours

**Given** un modèle LLM ou Whisper est téléchargé (fichier présent sur disque)
**When** la date de dernière utilisation est >= 15 jours dans le passé
**Then** le modèle est identifié comme "inutilisé"
**And** le seuil de 15 jours est une constante nommée `MODEL_INACTIVITY_THRESHOLD_DAYS = 15` (modifiable en code, pas par l'utilisateur)

**Given** un modèle n'a aucune clé `last_used` en AsyncStorage (modèles téléchargés avant cette story)
**When** le calcul d'inactivité est effectué
**Then** la date de téléchargement du fichier (via `FileSystem.getInfoAsync` stat) est utilisée comme fallback
**And** si le stat n'est pas disponible, le modèle n'est PAS considéré comme inutilisé (prudence → pas de faux positifs)

### AC4 : Alerte visuelle sur la carte modèle — LLM

**Given** un modèle LLM est identifié comme inutilisé (>= 15 jours)
**When** l'utilisateur navigue vers `LLMSettingsScreen`
**Then** la carte du modèle affiche un badge orange/ambre avec une icône d'avertissement
**And** un message s'affiche : "Non utilisé depuis X jours" (X = nombre de jours réels)
**And** deux boutons sont visibles :
  - "Supprimer (Y Go)" (Y = taille du modèle formatée en Go/Mo)
  - "Ignorer"
**And** l'alerte n'est visible que si le modèle est bien téléchargé (fichier présent)

### AC5 : Alerte visuelle sur la carte modèle — Whisper

**Given** un modèle Whisper est identifié comme inutilisé (>= 15 jours)
**When** l'utilisateur navigue vers `WhisperSettingsScreen`
**Then** la même alerte visuelle s'affiche sur la carte du modèle (cohérence avec AC4)

### AC6 : Action "Supprimer" depuis l'alerte

**Given** l'alerte d'inactivité est affichée sur une carte modèle
**When** l'utilisateur appuie sur "Supprimer (Y Go)"
**Then** une confirmation s'affiche : "Supprimer [Nom du modèle] (Y Go) ?"
**And** si confirmé → le fichier est supprimé (`deleteModel()`)
**And** les clés AsyncStorage du modèle sont nettoyées (lastUsed, dismissedAt)
**And** l'UI est mise à jour (carte revient en état "Non téléchargé")
**And** l'espace libéré est loggué en debug

### AC7 : Action "Ignorer" depuis l'alerte

**Given** l'alerte d'inactivité est affichée sur une carte modèle
**When** l'utilisateur appuie sur "Ignorer"
**Then** la clé `@pensieve/model_suggestion_dismissed_llm_${modelId}` (ou whisper) est créée avec le timestamp actuel
**And** l'alerte disparaît de la carte immédiatement
**And** l'alerte ne réapparaît plus sur les prochaines visites de l'écran — sauf si le modèle est réutilisé puis atteint à nouveau 15 jours d'inactivité

### AC8 : Tests unitaires et BDD

**Given** l'implémentation est complète
**When** les tests sont exécutés
**Then** les scénarios BDD dans `story-8-8-suggestion-suppression-modeles-inutilises.feature` couvrent :
  - AC1/AC2 : lastUsed mis à jour au téléchargement et à la sélection
  - AC3 : détection correcte des modèles inactifs (15 jours)
  - AC3 : modèle sans lastUsed (fallback comportement prudent)
  - AC6 : suppression depuis l'alerte
  - AC7 : dismissal persiste correctement
**And** les tests unitaires de `ModelUsageTrackingService` couvrent :
  - `trackModelUsed()` — persistance timestamp correct
  - `getUnusedModels()` — seuil 15 jours respecté
  - `dismissSuggestion()` — clé AsyncStorage créée
  - `hasDismissedSuggestion()` — retourne true si dismissedAt existe et modèle non réutilisé depuis
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression

## Tasks / Subtasks

### Task 1 : Créer `ModelUsageTrackingService` (AC1, AC2, AC3, AC7)

- [x] Subtask 1.1 : Créer l'interface `mobile/src/contexts/Normalization/domain/IModelUsageTrackingService.ts` :
  ```typescript
  export type ModelType = 'llm' | 'whisper';

  export interface UnusedModel {
    modelId: string;
    modelType: ModelType;
    daysSinceLastUse: number;
    sizeBytes?: number;
  }

  export interface IModelUsageTrackingService {
    trackModelUsed(modelId: string, modelType: ModelType): Promise<Result<void>>;
    getLastUsedDate(modelId: string, modelType: ModelType): Promise<Result<Date | null>>;
    getUnusedModels(
      downloadedLLMIds: string[],
      downloadedWhisperSizes: string[],
      thresholdDays?: number
    ): Promise<Result<UnusedModel[]>>;
    dismissSuggestion(modelId: string, modelType: ModelType): Promise<Result<void>>;
    hasDismissedSuggestion(modelId: string, modelType: ModelType): Promise<Result<boolean>>;
    clearModelTracking(modelId: string, modelType: ModelType): Promise<Result<void>>;
  }
  ```

- [x] Subtask 1.2 : Créer `mobile/src/contexts/Normalization/services/ModelUsageTrackingService.ts` :
  ```typescript
  // Constante centrale
  export const MODEL_INACTIVITY_THRESHOLD_DAYS = 15;

  // AsyncStorage keys (pattern : @pensieve/model_{action}_{type}_{id})
  const KEY_LAST_USED = (type: ModelType, id: string) =>
    `@pensieve/model_last_used_${type}_${id}`;
  const KEY_DISMISSED = (type: ModelType, id: string) =>
    `@pensieve/model_suggestion_dismissed_${type}_${id}`;
  ```
  - `trackModelUsed()` : `AsyncStorage.setItem(KEY_LAST_USED(...), new Date().toISOString())`
  - `getLastUsedDate()` : parser la clé, retourner `null` si absent
  - `getUnusedModels()` : pour chaque modèle téléchargé, vérifier si `daysSince(lastUsed) >= thresholdDays` ET `!hasDismissedSuggestion()`
  - `dismissSuggestion()` : `AsyncStorage.setItem(KEY_DISMISSED(...), new Date().toISOString())`
  - `hasDismissedSuggestion()` : retourner `true` si clé présente ET `lastUsedDate < dismissedDate` (i.e., pas réutilisé depuis le dismiss)
  - `clearModelTracking()` : supprimer les deux clés (appelé lors de `deleteModel()`)
  - Toutes les méthodes retournent `Result<T>` (ADR-023)

- [x] Subtask 1.3 : Ajouter le token dans `mobile/src/infrastructure/di/tokens.ts` :
  ```typescript
  IModelUsageTrackingService: Symbol.for('IModelUsageTrackingService'),
  ```

- [x] Subtask 1.4 : Enregistrer dans `mobile/src/infrastructure/di/container.ts` comme **Transient** (ADR-021) :
  ```typescript
  container.register<IModelUsageTrackingService>(
    TOKENS.IModelUsageTrackingService,
    { useClass: ModelUsageTrackingService }
  );
  ```

### Task 2 : Intégrer le tracking dans `LLMModelService` (AC1)

- [x] Subtask 2.1 : Injecter `IModelUsageTrackingService` dans le constructeur de `LLMModelService`
- [x] Subtask 2.2 : Dans le handler `.done()` de `downloadModel()`, appeler :
  ```typescript
  await this.usageTrackingService.trackModelUsed(modelId, 'llm');
  ```
- [x] Subtask 2.3 : Dans `setModelForTask()`, appeler `trackModelUsed(modelId, 'llm')` après la persistance
- [x] Subtask 2.4 : Dans `deleteModel()`, appeler `clearModelTracking(modelId, 'llm')` après suppression du fichier
- [x] Subtask 2.5 : Exposer une méthode `getDownloadedModelIds(): Promise<string[]>` (liste des modelId avec fichier sur disque) — nécessaire pour `getUnusedModels()` dans le screen store

### Task 3 : Intégrer le tracking dans `TranscriptionModelService` (AC2)

- [ ] Subtask 3.1 : Injecter `IModelUsageTrackingService` dans le constructeur
- [ ] Subtask 3.2 : Dans le handler de fin de téléchargement, appeler `trackModelUsed(modelSize, 'whisper')`
- [ ] Subtask 3.3 : Dans `setSelectedModel()`, appeler `trackModelUsed(modelSize, 'whisper')`
- [ ] Subtask 3.4 : Dans `deleteModel()`, appeler `clearModelTracking(modelSize, 'whisper')`
- [ ] Subtask 3.5 : Exposer `getDownloadedModelSizes(): Promise<string[]>` — liste des tailles Whisper avec fichier présent

### Task 4 : Mettre à jour `LLMModelCard` — alerte visuelle (AC4, AC6, AC7)

- [ ] Subtask 4.1 : Ajouter des props à `LLMModelCard` :
  ```typescript
  interface LLMModelCardProps {
    // ... props existantes ...
    unusedDays?: number;           // si défini > 0 = modèle inactif
    onDeleteUnused?: () => void;   // callback suppression depuis alerte
    onDismissUnused?: () => void;  // callback ignorer l'alerte
  }
  ```
- [ ] Subtask 4.2 : Implémenter le bloc d'alerte conditionnel (visible uniquement si `unusedDays >= 15`) :
  - Badge orange/ambre avec icône `⚠️` ou similaire
  - Texte : `"Non utilisé depuis ${unusedDays} jours"`
  - Bouton primaire : `"Supprimer (${formattedSize})"` → `onDeleteUnused()`
  - Bouton secondaire : `"Ignorer"` → `onDismissUnused()`
- [ ] Subtask 4.3 : La confirmation avant suppression peut être gérée au niveau du Screen (pas dans la carte)
- [ ] Subtask 4.4 : **NE PAS** modifier l'interface existante des props (backward compatible — `unusedDays` optionnel)

### Task 5 : Mettre à jour `WhisperModelCard` — alerte visuelle (AC5, AC6, AC7)

- [ ] Subtask 5.1 : Même ajout de props que `LLMModelCard` (cohérence)
- [ ] Subtask 5.2 : Même bloc d'alerte conditionnel

### Task 6 : Mettre à jour les screens (AC4, AC5, AC6, AC7)

- [ ] Subtask 6.1 : Dans `LLMSettingsScreen`, au montage et au focus :
  ```typescript
  useEffect(() => {
    const checkUnusedModels = async () => {
      const downloadedIds = await llmModelService.getDownloadedModelIds();
      const result = await usageTrackingService.getUnusedModels(downloadedIds, [], MODEL_INACTIVITY_THRESHOLD_DAYS);
      if (result.type === ResultType.SUCCESS) {
        setUnusedLLMModels(result.value);
      }
    };
    checkUnusedModels();
  }, []);
  ```
- [ ] Subtask 6.2 : Passer `unusedDays` à chaque `LLMModelCard` (0 si pas dans la liste unusedModels)
- [ ] Subtask 6.3 : Implémenter `handleDeleteUnused(modelId)` :
  - Alert de confirmation : `Alert.alert("Supprimer le modèle ?", "...", [{text:"Annuler"}, {text:"Supprimer", onPress:...}])`
  - Si confirmé : `llmModelService.deleteModel(modelId)` (qui appellera `clearModelTracking` — Task 2.4)
  - Rafraîchir la liste des modèles
- [ ] Subtask 6.4 : Implémenter `handleDismissUnused(modelId)` :
  - `usageTrackingService.dismissSuggestion(modelId, 'llm')`
  - Retirer le modèle de `unusedLLMModels` (mise à jour état local)
- [ ] Subtask 6.5 : Même logique dans `WhisperSettingsScreen` (avec `'whisper'` et `downloadedModelSizes`)

### Task 7 : Tests BDD (AC8)

- [ ] Subtask 7.1 : Créer `mobile/tests/acceptance/features/story-8-8-suggestion-suppression-modeles-inutilises.feature`
- [ ] Subtask 7.2 : Écrire les scénarios (voir section "Scénarios BDD" ci-dessous)
- [ ] Subtask 7.3 : Créer `mobile/tests/acceptance/story-8-8-suggestion-suppression-modeles-inutilises.test.ts` avec step definitions
- [ ] Subtask 7.4 : Mocker `ModelUsageTrackingService`, `LLMModelService`, `TranscriptionModelService`

### Task 8 : Tests unitaires — `ModelUsageTrackingService` (AC8)

- [ ] Subtask 8.1 : Créer `mobile/src/contexts/Normalization/services/__tests__/ModelUsageTrackingService.test.ts`
- [ ] Subtask 8.2 : Cas testés :
  ```
  Cas 1 — trackModelUsed → AsyncStorage.setItem appelé avec timestamp ISO valide
  Cas 2 — trackModelUsed → retourne Result SUCCESS
  Cas 3 — getLastUsedDate → retourne la date parsée si clé présente
  Cas 4 — getLastUsedDate → retourne null si clé absente
  Cas 5 — getUnusedModels — modèle utilisé il y a 14 jours → PAS dans la liste
  Cas 6 — getUnusedModels — modèle utilisé il y a 16 jours → DANS la liste
  Cas 7 — getUnusedModels — modèle sans lastUsed et sans stat → PAS dans la liste (prudence)
  Cas 8 — dismissSuggestion → KEY_DISMISSED créée avec timestamp
  Cas 9 — hasDismissedSuggestion → true si dismissed et pas réutilisé depuis
  Cas 10 — hasDismissedSuggestion → false si dismissed mais modèle réutilisé après dismiss
  Cas 11 — clearModelTracking → supprime KEY_LAST_USED et KEY_DISMISSED
  ```
- [ ] Subtask 8.3 : Mock de `AsyncStorage` via `@react-native-async-storage/async-storage/jest/async-storage-mock`

### Task 9 : Validation finale (AC8)

- [ ] Subtask 9.1 : `npm run test:unit` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 9.2 : `npm run test:acceptance` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 9.3 : Test manuel — télécharger un modèle, changer la date lastUsed en AsyncStorage (via debug) à J-16, relancer l'app, vérifier que le badge apparaît
- [ ] Subtask 9.4 : Fermer l'issue GitHub #8 avec référence au commit

## Scénarios BDD (Feature File)

```gherkin
# language: fr
Fonctionnalité: Suggestion de suppression des modèles inutilisés
  En tant qu'utilisateur ayant téléchargé plusieurs modèles
  Je veux être alerté quand un modèle n'a pas été utilisé depuis 15 jours
  Afin de libérer de l'espace disque en supprimant les modèles inutiles

  Contexte:
    Étant donné que je suis un utilisateur authentifié
    Et que le service ModelUsageTrackingService est initialisé

  Scénario: Date de dernière utilisation initialisée au téléchargement LLM
    Étant donné que le modèle LLM "qwen2.5-0.5b" n'est pas téléchargé
    Quand le téléchargement du modèle se termine avec succès
    Alors la date de dernière utilisation est enregistrée avec la date actuelle
    Et la clé "@pensieve/model_last_used_llm_qwen2.5-0.5b" existe en AsyncStorage

  Scénario: Date de dernière utilisation mise à jour à la sélection
    Étant donné que le modèle LLM "qwen2.5-0.5b" a une lastUsed date de il y a 10 jours
    Quand l'utilisateur sélectionne le modèle "qwen2.5-0.5b" pour une tâche
    Alors la date de dernière utilisation est mise à jour à la date actuelle
    Et le modèle n'apparaît plus dans la liste des modèles inutilisés

  Scénario: Modèle LLM identifié comme inutilisé après 15 jours
    Étant donné que le modèle LLM "qwen2.5-3b" est téléchargé sur le disque
    Et que sa date de dernière utilisation est il y a 16 jours
    Et que l'alerte n'a pas été ignorée
    Quand le système vérifie les modèles inutilisés
    Alors le modèle "qwen2.5-3b" est retourné dans la liste des modèles inactifs
    Et le nombre de jours est "16"

  Scénario: Modèle LLM non identifié si utilisé il y a 14 jours
    Étant donné que le modèle LLM "smollm-135m" est téléchargé sur le disque
    Et que sa date de dernière utilisation est il y a 14 jours
    Quand le système vérifie les modèles inutilisés
    Alors le modèle "smollm-135m" n'est pas dans la liste des modèles inactifs

  Scénario: Modèle sans lastUsed — comportement prudent
    Étant donné que le modèle LLM "qwen2.5-3b" est téléchargé sur le disque
    Et qu'aucune clé "lastUsed" n'existe pour ce modèle
    Et que le stat du fichier n'est pas disponible
    Quand le système vérifie les modèles inutilisés
    Alors le modèle "qwen2.5-3b" n'est PAS dans la liste des modèles inactifs

  Scénario: Suppression d'un modèle depuis l'alerte
    Étant donné que le modèle "qwen2.5-3b" affiche une alerte d'inactivité
    Quand l'utilisateur confirme la suppression
    Alors le fichier du modèle est supprimé du disque
    Et les clés AsyncStorage du modèle sont supprimées
    Et la carte du modèle passe en état "Non téléchargé"

  Scénario: Ignorer une alerte — ne réapparaît pas
    Étant donné que le modèle "qwen2.5-3b" affiche une alerte d'inactivité
    Quand l'utilisateur appuie sur "Ignorer"
    Alors la clé de dismissal est créée en AsyncStorage
    Et le modèle ne réapparaît pas dans la liste des modèles inactifs lors des visites suivantes
```

## Dev Notes

### 🔑 Fichiers à modifier

```
pensieve/mobile/src/contexts/Normalization/services/LLMModelService.ts
  ← Task 2: Injecter IModelUsageTrackingService
  ← Task 2: trackModelUsed() dans done handler + setModelForTask()
  ← Task 2: clearModelTracking() dans deleteModel()
  ← Task 2: Exposer getDownloadedModelIds()

pensieve/mobile/src/contexts/Normalization/services/TranscriptionModelService.ts
  ← Task 3: Idem pour Whisper

pensieve/mobile/src/components/llm/LLMModelCard.tsx
  ← Task 4: Props unusedDays + onDeleteUnused + onDismissUnused
  ← Task 4: Bloc alerte conditionnel

pensieve/mobile/src/components/whisper/WhisperModelCard.tsx
  ← Task 5: Idem

pensieve/mobile/src/screens/settings/LLMSettingsScreen.tsx
  ← Task 6: Calcul unused models au montage, confirmation suppression

pensieve/mobile/src/screens/settings/WhisperSettingsScreen.tsx
  ← Task 6: Idem

pensieve/mobile/src/infrastructure/di/container.ts
  ← Task 1: Enregistrer ModelUsageTrackingService (Transient)

pensieve/mobile/src/infrastructure/di/tokens.ts
  ← Task 1: Ajouter IModelUsageTrackingService token
```

### 📁 Fichiers à créer

```
pensieve/mobile/src/contexts/Normalization/domain/IModelUsageTrackingService.ts
  ← Task 1.1: Interface + types ModelType + UnusedModel

pensieve/mobile/src/contexts/Normalization/services/ModelUsageTrackingService.ts
  ← Task 1.2: Implémentation avec constante MODEL_INACTIVITY_THRESHOLD_DAYS

pensieve/mobile/src/contexts/Normalization/services/__tests__/ModelUsageTrackingService.test.ts
  ← Task 8: Tests unitaires (11 cas)

pensieve/mobile/tests/acceptance/features/story-8-8-suggestion-suppression-modeles-inutilises.feature
  ← Task 7: Scénarios Gherkin

pensieve/mobile/tests/acceptance/story-8-8-suggestion-suppression-modeles-inutilises.test.ts
  ← Task 7: Step definitions BDD
```

### ⛔ Fichiers à NE PAS modifier

```
pensieve/mobile/src/contexts/Normalization/services/llmModelsConfig.ts
  ← Configuration statique des modèles — pas de changement requis

pensieve/mobile/src/stores/settingsStore.ts
  ← Store global — le tracking lastUsed est géré dans le service dédié, pas le store global

pensieve/mobile/src/stores/llmSettingsScreenStore.ts
  ← Store UI-spécifique — les unused models peuvent être gérés localement dans le Screen
    via useState (pas de persistance nécessaire pour cet état dérivé)
```

### 🏗️ Architecture — Flux complet

```
Au téléchargement (done handler):
  LLMModelService.downloadModel() / TranscriptionModelService.downloadModel()
    └─ ModelUsageTrackingService.trackModelUsed(modelId, 'llm'|'whisper')
         └─ AsyncStorage.setItem('@pensieve/model_last_used_llm_${modelId}', isoTimestamp)

À la sélection:
  LLMModelService.setModelForTask() / TranscriptionModelService.setSelectedModel()
    └─ ModelUsageTrackingService.trackModelUsed(modelId, type)

À la suppression:
  LLMModelService.deleteModel() / TranscriptionModelService.deleteModel()
    └─ ModelUsageTrackingService.clearModelTracking(modelId, type)
         └─ AsyncStorage.removeItem(KEY_LAST_USED + KEY_DISMISSED)

Au montage des Settings screens:
  LLMSettingsScreen / WhisperSettingsScreen
    ├─ llmModelService.getDownloadedModelIds()
    └─ usageTrackingService.getUnusedModels(ids, sizes, 15)
         └─ Pour chaque modèle : daysSince(lastUsed) >= 15 && !hasDismissed
              → UnusedModel[] → prop unusedDays vers LLMModelCard/WhisperModelCard

Action "Ignorer":
  usageTrackingService.dismissSuggestion(modelId, type)
    └─ AsyncStorage.setItem('@pensieve/model_suggestion_dismissed_${type}_${id}', isoTimestamp)

Action "Supprimer":
  Alert.alert() → confirm → llmModelService.deleteModel()
    └─ clearModelTracking() → suppression fichier + AsyncStorage cleanup
```

### 📋 Conformité ADR

| ADR | Impact | Décision |
|-----|--------|----------|
| **ADR-021** (DI Lifecycle — Transient First) | `ModelUsageTrackingService` : pas de cache technique, stateless → **Transient** | ✅ `container.register()` |
| **ADR-022** (OP-SQLite pour état critique) | `lastUsed` date = métadonnée UI/préférence non-critique, pattern AsyncStorage déjà utilisé dans les services existants (LLMModelService, TranscriptionModelService). Cohérence > migration de dette tech dans cette story. | ⚠️ AsyncStorage acceptable (cohérence avec existant, non-critique) |
| **ADR-023** (Result Pattern) | Toutes les méthodes de `ModelUsageTrackingService` retournent `Result<T>`. AsyncStorage = try/catch autorisé (external call) | ✅ |
| **ADR-024** (Clean Code / SRP) | Service dédié `ModelUsageTrackingService` = 1 responsabilité (tracking usage). LLMModelService pas gonflé davantage | ✅ |
| **ADR-018** (OP-SQLite) | Aucun impact — le tracking reste dans AsyncStorage (cohérence services existants) | ✅ |

### 🚨 Points de vigilance techniques

#### 1. `hasDismissedSuggestion()` — logique temporelle

La logique doit être :
```typescript
// Dismissed seulement si l'utilisateur a ignoré ET n'a pas réutilisé depuis
const dismissedDate = await getDate(KEY_DISMISSED);
const lastUsedDate = await getDate(KEY_LAST_USED);

if (!dismissedDate) return false; // pas de dismiss
if (!lastUsedDate) return true;   // dismissé, jamais réutilisé → toujours dismissé

// Si l'utilisateur a réutilisé le modèle APRÈS avoir ignoré → réinitialiser
return dismissedDate > lastUsedDate;
```

#### 2. Confirmation de suppression — Pattern Alert.alert

Utiliser le pattern existant dans WhisperSettingsScreen :
```typescript
Alert.alert(
  "Supprimer le modèle",
  `Supprimer ${modelName} (${formattedSize}) et libérer l'espace disque ?`,
  [
    { text: "Annuler", style: "cancel" },
    { text: "Supprimer", style: "destructive", onPress: () => deleteModel(modelId) }
  ]
);
```

#### 3. Stat du fichier pour fallback date

```typescript
// Fallback : si pas de lastUsed, utiliser la date de modification du fichier
const fileInfo = await FileSystem.getInfoAsync(modelPath, { size: true });
if (fileInfo.exists && fileInfo.modificationTime) {
  return new Date(fileInfo.modificationTime * 1000); // en secondes → ms
}
return null; // pas de stat → prudence, ne pas suggérer suppression
```

#### 4. Taille du modèle pour l'affichage

Pour afficher "Supprimer (2.0 Go)" :
```typescript
// LLMModelService : la taille est dans llmModelsConfig
const modelConfig = LLM_MODELS.find(m => m.id === modelId);
const formattedSize = formatBytes(modelConfig?.fileSize ?? 0);

// WhisperModelCard : tailles connues dans whisperModelsConfig
```

#### 5. LLMModelService — `getDownloadedModelIds()`

Cette méthode existe probablement déjà sous une autre forme. Vérifier l'interface `ILLMModelService` avant de l'ajouter. Si elle existe → ne pas dupliquer.

### 🎯 Stratégie de test

Les tests BDD simuleront via mocks :
1. `AsyncStorage` → mockée avec `@react-native-async-storage/async-storage/jest/async-storage-mock`
2. Dates passées → utiliser `jest.useFakeTimers()` ou passer directement des timestamps calculés
3. `FileSystem.getInfoAsync` → mockée pour les cas de fallback

Pattern de mock date recommandé :
```typescript
// Dans le test : forcer la date actuelle
const fixedNow = new Date('2026-03-15T10:00:00Z');
jest.spyOn(Date, 'now').mockReturnValue(fixedNow.getTime());

// Stocker en AsyncStorage une date 16 jours avant
const oldDate = new Date('2026-02-27T10:00:00Z').toISOString();
await AsyncStorage.setItem('@pensieve/model_last_used_llm_qwen2.5-3b', oldDate);
```

### Project Structure Notes

```
pensieve/mobile/src/
├── contexts/Normalization/
│   ├── domain/
│   │   ├── ILLMModelService.ts                        ← VÉRIFIER getDownloadedModelIds()
│   │   ├── ITranscriptionModelService.ts              ← VÉRIFIER getDownloadedModelSizes()
│   │   └── IModelUsageTrackingService.ts              ← CRÉER (Task 1.1)
│   └── services/
│       ├── LLMModelService.ts                         ← MODIFIER (Task 2)
│       ├── TranscriptionModelService.ts               ← MODIFIER (Task 3)
│       └── ModelUsageTrackingService.ts               ← CRÉER (Task 1.2)
├── components/
│   ├── llm/LLMModelCard.tsx                          ← MODIFIER (Task 4)
│   └── whisper/WhisperModelCard.tsx                  ← MODIFIER (Task 5)
├── screens/settings/
│   ├── LLMSettingsScreen.tsx                          ← MODIFIER (Task 6)
│   └── WhisperSettingsScreen.tsx                     ← MODIFIER (Task 6)
└── infrastructure/di/
    ├── container.ts                                   ← MODIFIER (Task 1.4)
    └── tokens.ts                                      ← MODIFIER (Task 1.3)

tests/acceptance/
├── features/
│   └── story-8-8-suggestion-suppression-modeles-inutilises.feature  ← CRÉER
└── story-8-8-suggestion-suppression-modeles-inutilises.test.ts       ← CRÉER

src/contexts/Normalization/services/__tests__/
└── ModelUsageTrackingService.test.ts                                  ← CRÉER
```

### References

- [Source: GitHub Issue #8](https://github.com/yohikofox/pensieve/issues/8) — Spécification originale
- [Source: `mobile/src/contexts/Normalization/services/LLMModelService.ts`] — Service LLM à modifier, pattern AsyncStorage existant
- [Source: `mobile/src/contexts/Normalization/services/TranscriptionModelService.ts`] — Service Whisper à modifier
- [Source: `mobile/src/components/llm/LLMModelCard.tsx`] — Composant à étendre
- [Source: `mobile/src/components/whisper/WhisperModelCard.tsx`] — Composant à étendre
- [Source: `mobile/src/infrastructure/di/tokens.ts`] — Pattern token DI
- [Source: `mobile/src/infrastructure/di/container.ts`] — Pattern enregistrement Transient
- [Source: `_bmad-output/implementation-artifacts/stories/epic-8/8-7-download-modeles-en-background.md`] — Story précédente : pattern ModelDownloadNotificationService (même BC Normalization, même pattern DI)
- [Source: ADR-021] — DI Lifecycle (Transient First)
- [Source: ADR-022] — Persistence (OP-SQLite préféré, AsyncStorage acceptable pour métadonnées non-critiques)
- [Source: ADR-023] — Result Pattern (8 types, try/catch uniquement pour external calls)
- [Source: ADR-024] — Clean Code (SRP, fonctions < 50 lignes, nommage révélateur)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

**Task 2 — Intégration du tracking dans `LLMModelService` (2026-03-01)**

- `IModelUsageTrackingService` injecté via `@inject(TOKENS.IModelUsageTrackingService)` dans le constructeur (Subtask 2.1)
- `trackModelUsed(modelId, 'llm')` appelé dans le handler `.done()` de `downloadModel()` : initialise `lastUsed` dès la fin du téléchargement (AC1)
- `trackModelUsed(modelId, 'llm')` appelé dans `setModelForTask()` (branche `else`, quand `modelId !== null`) : met à jour `lastUsed` à chaque sélection (AC1)
- `clearModelTracking(modelId, 'llm')` appelé dans `deleteModel()` après suppression fichier : nettoie les clés AsyncStorage du modèle (AC6)
- `getDownloadedModelIds(): Promise<string[]>` ajouté à `ILLMModelService` (interface) et implémenté dans `LLMModelService` (délègue à `getDownloadedModels()`) — Subtask 2.5
- Tous les appels au tracking sont fire-and-forget (`.catch(() => {})`) pour ne pas bloquer le flux principal
- 11/11 tests `ModelUsageTrackingService` passent, 0 régression, TypeScript conforme

**Task 1 — Implémentation de `ModelUsageTrackingService` (2026-03-01)**

- Interface `IModelUsageTrackingService` créée avec 6 méthodes + types `ModelType` et `UnusedModel`
- Implémentation `ModelUsageTrackingService` : service stateless, `@injectable()`, toutes méthodes retournent `Result<T>` (ADR-023)
- Constante `MODEL_INACTIVITY_THRESHOLD_DAYS = 15` (AC3)
- Pattern AsyncStorage : `@pensieve/model_last_used_{type}_{id}` et `@pensieve/model_suggestion_dismissed_{type}_{id}`
- Logique `hasDismissedSuggestion()` : temporelle — dismissed seulement si `dismissedDate > lastUsedDate` (réinitialisation implicite à la réutilisation)
- Comportement prudent dans `getUnusedModels()` : si pas de lastUsed → non inclus (évite faux positifs pour modèles pré-story)
- Token `IModelUsageTrackingService` ajouté dans `tokens.ts` via `Symbol.for()`
- Enregistrement Transient dans `container.ts` (ADR-021 — service stateless sans état partagé)
- 11/11 tests unitaires passent (0 régression sur la suite complète préexistante)

### File List

**Fichiers créés (Task 1) :**
- `pensieve/mobile/src/contexts/Normalization/domain/IModelUsageTrackingService.ts`
- `pensieve/mobile/src/contexts/Normalization/services/ModelUsageTrackingService.ts`
- `pensieve/mobile/src/contexts/Normalization/services/__tests__/ModelUsageTrackingService.test.ts`

**Fichiers modifiés (Task 1) :**
- `pensieve/mobile/src/infrastructure/di/tokens.ts` — ajout token `IModelUsageTrackingService`
- `pensieve/mobile/src/infrastructure/di/container.ts` — enregistrement Transient

**Fichiers modifiés (Task 2) :**
- `pensieve/mobile/src/contexts/Normalization/domain/ILLMModelService.ts` — ajout `getDownloadedModelIds(): Promise<string[]>`
- `pensieve/mobile/src/contexts/Normalization/services/LLMModelService.ts` — injection `IModelUsageTrackingService`, tracking dans `.done()` + `setModelForTask()` + `deleteModel()`, implémentation `getDownloadedModelIds()`

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #8 — analyse exhaustive : aucun tracking lastUsed existant dans LLMModelService ni TranscriptionModelService, service dédié ModelUsageTrackingService créé (SRP ADR-024), AsyncStorage pour cohérence avec les services existants (note ADR-022), Result Pattern ADR-023, Transient ADR-021. UI : extension props LLMModelCard + WhisperModelCard, confirmation Alert.alert, dismiss persisté. | yohikofox |
| 2026-03-01 | Task 1 implémentée — IModelUsageTrackingService + ModelUsageTrackingService + token DI + enregistrement container Transient. 11/11 tests unitaires verts. | yohikofox |
| 2026-03-01 | Task 2 implémentée — Intégration tracking dans LLMModelService : injection DI, trackModelUsed au téléchargement + sélection, clearModelTracking à la suppression, getDownloadedModelIds() exposé. 0 régression, TypeScript conforme. | yohikofox |
