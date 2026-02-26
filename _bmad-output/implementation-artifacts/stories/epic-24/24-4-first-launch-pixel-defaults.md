# Story 24.4: First Launch Initializer — Défauts Automatiques sur Pixel 9+

Status: done

## Story

As a **utilisateur sur un appareil Google Pixel 9 ou supérieur**,
I want **que l'application configure automatiquement les options optimales pour mon appareil au premier lancement**,
So that **je bénéficie immédiatement de la reconnaissance native et du modèle IA le plus adapté sans avoir à naviguer dans les settings**.

**Context:** Sur Pixel 9+ (Google Tensor G4+), la reconnaissance vocale native est supérieure à Whisper pour la latence et ne requiert pas de téléchargement. Gemma 3 1B MediaPipe est optimisé pour le Tensor TPU et offre de meilleures performances que les modèles génériques. Cette story crée un `FirstLaunchInitializer` qui détecte l'appareil une seule fois après la première connexion et applique les paramètres optimaux.

## Acceptance Criteria

### AC1: Détection Premier Lancement
**Given** l'utilisateur vient de se connecter pour la première fois
**When** `MainApp.tsx` s'initialise après auth réussie
**Then** `FirstLaunchInitializer.run()` est appelé
**And** l'initializer vérifie `AsyncStorage.getItem('@pensieve/first_launch_completed')`
**And** si la clé n'existe pas → exécuter l'initialisation (AC2-AC6)
**And** si la clé existe avec valeur `'true'` → ne rien faire

### AC2: Détection Pixel 9+
**Given** l'initialisation premier lancement est en cours
**When** `NPUDetectionService.detectNPU()` est appelé
**Then** l'appareil est reconnu comme Pixel 9+ si `manufacturer === 'Google'` ET `model` correspond à un des patterns : `Pixel 9`, `Pixel 9 Pro`, `Pixel 9 Pro XL`, `Pixel 9 Pro Fold`, `Pixel 9a`
**And** tout Pixel 10+ ultérieur (détecté via `manufacturer === 'Google'` ET `chipset === 'Tensor G5'` ou supérieur) est également inclus
**And** les Pixel 6-8 ne sont pas concernés par cette initialisation automatique

### AC3: Activation de la Reconnaissance Native
**Given** l'appareil est identifié comme Pixel 9+
**When** l'initialisation s'exécute
**Then** `TranscriptionEngineService.setSelectedEngineType('native')` est appelé
**And** le setting est persisté via AsyncStorage (`@pensieve/transcription_engine = 'native'`)

### AC4: Activation de la Transcription Automatique
**Given** l'appareil est identifié comme Pixel 9+
**When** l'initialisation s'exécute
**Then** `settingsStore.setAutoTranscriptionEnabled(true)` est appelé
**And** le setting est persisté (déjà géré par le store Zustand)

### AC5: Téléchargement et Configuration de Gemma 3 1B MediaPipe
**Given** l'appareil est identifié comme Pixel 9+
**When** l'initialisation s'exécute
**Then** `LLMModelService.downloadModel('gemma3-1b-mediapipe', onProgress)` est déclenché
**And** une fois le téléchargement terminé, `LLMModelService.setModelForTask('postProcessing', 'gemma3-1b-mediapipe')` est appelé
**And** `LLMModelService.setModelForTask('analysis', 'gemma3-1b-mediapipe')` est appelé
**And** si le modèle est déjà téléchargé, le téléchargement est skippé mais l'assignation est faite

### AC6: UI de Progression — Premier Lancement
**Given** le téléchargement de Gemma 3 1B MediaPipe est en cours
**When** l'utilisateur voit l'écran après connexion
**Then** un écran (ou bottom sheet) de bienvenue s'affiche avec :
  - Message : "Optimisation pour votre Pixel 9"
  - Barre de progression du téléchargement
  - Taille estimée du modèle
  - Possibilité de skip (l'initialisation se termine en background)
**And** si l'utilisateur skippe, le téléchargement continue en background via `react-native-background-downloader`
**And** une fois le téléchargement terminé (avec ou sans skip), l'écran disparaît

### AC7: Marquage Fin d'Initialisation
**Given** l'initialisation est terminée (téléchargement fini OU skippé)
**When** `FirstLaunchInitializer.run()` se termine
**Then** `AsyncStorage.setItem('@pensieve/first_launch_completed', 'true')` est appelé
**And** les prochains lancements de l'app ignorent complètement l'initialisation (AC1)

### AC8: Appareils Non-Pixel
**Given** l'appareil n'est pas un Pixel 9+
**When** l'initialisation premier lancement s'exécute
**Then** aucun setting n'est modifié automatiquement (experience par défaut)
**And** `@pensieve/first_launch_completed` est quand même marqué `'true'`
**And** aucune UI de progression n'est affichée

### AC9: Robustesse — Échec de Téléchargement
**Given** le téléchargement de Gemma échoue (réseau, espace insuffisant)
**When** `LLMModelService.downloadModel()` rejette
**Then** l'erreur est logguée silencieusement (pas de crash)
**And** les settings transcription (AC3, AC4) sont quand même persistés
**And** `@pensieve/first_launch_completed` est marqué `'true'` (pas de boucle infinie)
**And** l'utilisateur peut réessayer le téléchargement depuis les Settings LLM

## Tasks / Subtasks

### Mobile — Service

- [x] **Task 1: FirstLaunchInitializer Service** (AC1-AC9)
  - [x] Subtask 1.1: Créer `mobile/src/contexts/identity/services/FirstLaunchInitializer.ts`
    - Méthode `run(): Promise<void>`
    - Injecte : `NPUDetectionService`, `TranscriptionEngineService`, `LLMModelService`, `settingsStore`
    - Logique : check → detect → apply → mark
  - [x] Subtask 1.2: Constante AsyncStorage key : `@pensieve/first_launch_completed`
  - [x] Subtask 1.3: Détection Pixel 9+ — liste des modèles à matcher (AC2)
    ```
    'Pixel 9', 'Pixel 9 Pro', 'Pixel 9 Pro XL', 'Pixel 9 Pro Fold', 'Pixel 9a',
    'Pixel 10', 'Pixel 10 Pro', ... (pattern : manufacturer=Google + Tensor G4+)
    ```
  - [x] Subtask 1.4: Gestion erreurs téléchargement (catch silencieux + log)
  - [x] Subtask 1.5: Enregistrer `FirstLaunchInitializer` dans `container.ts` (singleton)

- [x] **Task 2: Enregistrement DI** (AC1)
  - [x] Subtask 2.1: Token DI dans `tokens.ts`
  - [x] Subtask 2.2: Enregistrement dans `container.ts`

### Mobile — Intégration MainApp

- [x] **Task 3: Appel dans MainApp.tsx** (AC1)
  - [x] Subtask 3.1: Résoudre `FirstLaunchInitializer` depuis le container après auth réussie
  - [x] Subtask 3.2: Appeler `initializer.run()` une fois l'utilisateur authentifié
  - [x] Subtask 3.3: Passer le callback `onProgress` à l'UI (via state ou context)

### Mobile — UI

- [x] **Task 4: Composant FirstLaunchScreen / BottomSheet** (AC6)
  - [x] Subtask 4.1: Créer composant `FirstLaunchProgress` (modal overlay)
  - [x] Subtask 4.2: Afficher : titre "Optimisation pour votre Pixel 9" + barre de progression + taille modèle
  - [x] Subtask 4.3: Bouton "Continuer en arrière-plan" (skip)
  - [x] Subtask 4.4: Auto-dismiss quand téléchargement terminé
  - [x] Subtask 4.5: Ne pas afficher sur appareils non-Pixel (AC8)

### Tests

- [x] **Task 5: Tests** (AC1-AC9)
  - [x] Subtask 5.1: Tests unitaires `FirstLaunchInitializer`
    - Test : premier lancement → run exécute les étapes
    - Test : `first_launch_completed = 'true'` → run ne fait rien
    - Test : Pixel 9 détecté → AC3 + AC4 + AC5 appelés
    - Test : Non-Pixel → aucun setting modifié, marqué completed
    - Test : échec download → settings quand même persistés + marqué completed
    - Test : modèle déjà téléchargé → download skippé, assignation faite
  - [x] Subtask 5.2: Tests BDD `story-24-4.feature`
    - Scénario: premier lancement Pixel 9 → native recognition activé
    - Scénario: second lancement → initializer no-op
    - Scénario: premier lancement Samsung → pas de changement

## Technical Notes

### Structure FirstLaunchInitializer

```typescript
@injectable()
export class FirstLaunchInitializer {
  constructor(
    @inject(TOKENS.NPUDetectionService) private npu: NPUDetectionService,
    @inject(TOKENS.TranscriptionEngineService) private engine: TranscriptionEngineService,
    @inject(TOKENS.LLMModelService) private llm: LLMModelService,
  ) {}

  async run(onProgress?: (progress: number) => void): Promise<void> {
    const alreadyDone = await AsyncStorage.getItem('@pensieve/first_launch_completed');
    if (alreadyDone === 'true') return;

    try {
      const npu = await this.npu.detectNPU();
      if (this.isPixel9Plus(npu)) {
        await this.engine.setSelectedEngineType('native');
        useSettingsStore.getState().setAutoTranscriptionEnabled(true);
        await this.downloadGemmaWithFallback(onProgress);
      }
    } catch (error) {
      // Log silencieux — jamais de crash sur initialisation
      console.warn('[FirstLaunchInitializer] Error:', error);
    } finally {
      await AsyncStorage.setItem('@pensieve/first_launch_completed', 'true');
    }
  }

  private isPixel9Plus(npu: NPUInfo): boolean {
    const PIXEL_9_PLUS_MODELS = [
      'Pixel 9', 'Pixel 9 Pro', 'Pixel 9 Pro XL', 'Pixel 9 Pro Fold', 'Pixel 9a',
    ];
    return (
      npu.manufacturer === 'Google' &&
      PIXEL_9_PLUS_MODELS.some(m => npu.model?.startsWith(m))
    ) || (npu.manufacturer === 'Google' && npu.chipsetGeneration >= 4); // Tensor G4+
  }

  private async downloadGemmaWithFallback(onProgress?: (n: number) => void): Promise<void> {
    const MODEL_ID = 'gemma3-1b-mediapipe';
    try {
      const alreadyDownloaded = await this.llm.isModelDownloaded(MODEL_ID);
      if (!alreadyDownloaded) {
        await this.llm.downloadModel(MODEL_ID, onProgress);
      }
      await this.llm.setModelForTask('postProcessing', MODEL_ID);
      await this.llm.setModelForTask('analysis', MODEL_ID);
    } catch (error) {
      console.warn('[FirstLaunchInitializer] Gemma download failed:', error);
      // Non-bloquant
    }
  }
}
```

### Clé AsyncStorage

```
@pensieve/first_launch_completed : 'true' | null
```

Cette clé est volontairement simple (pas de JSON) pour minimiser le risque de corruption.

### Modèle Gemma

L'ID `'gemma3-1b-mediapipe'` doit correspondre exactement à la clé dans `llmModelsConfig.ts`. À vérifier au moment de l'implémentation.

## Dependencies

- `NPUDetectionService` (existant — Pixel 9 déjà mappé dans `NPUDetectionService`)
- `LLMModelService` (existant — `downloadModel`, `setModelForTask`, `isModelDownloaded`)
- `TranscriptionEngineService` (existant — `setSelectedEngineType`)
- `settingsStore` (existant — `setAutoTranscriptionEnabled`)
- Story 24.3 non requise (peut se développer indépendamment)

## Definition of Done

- [x] `FirstLaunchInitializer` service créé et enregistré en DI
- [x] Appelé dans `MainApp.tsx` après auth
- [x] Pixel 9+ détecté → native recognition + auto-transcription + Gemma configurés
- [x] Non-Pixel → marqué completed sans modification
- [x] UI progress visible sur Pixel 9+ pendant téléchargement
- [x] Skip possible (background download)
- [x] Second lancement → no-op garanti
- [x] Échec download → non-bloquant
- [x] Tests unitaires (≥ 6 cas) passent — **14/14 (6 cas + 8 cas limites isPixel9Plus)**
- [x] Tests BDD (3 scénarios) passent — **3/3**

## Dev Notes

### Architecture Compliance

- **ADR-021 (Transient First — exception justifiée)** : `FirstLaunchInitializer` doit être enregistré comme **singleton** dans `container.ts` (`registerSingleton`). Justification : le service cache en mémoire le flag `alreadyDone` pour éviter les re-lectures AsyncStorage répétées entre renders. C'est une exception explicitement documentée à la règle "transient first".
- **DI Pattern** : Utiliser `@injectable()` + `@inject(TOKENS.X)` pour chaque dépendance injectée. Déclarer un token `TOKENS.FirstLaunchInitializer` dans `tokens.ts`. Pattern référence : voir `NPUDetectionService` dans `container.ts`.
- **Cross-context dependency** : `NPUDetectionService` est dans le contexte `Normalization` (`mobile/src/contexts/Normalization/services/`), pas dans `identity`. Acceptable dans le container DI (le container est cross-context par design).
- **AsyncStorage clé** : `@pensieve/first_launch_completed` — valeur string `'true'` | `null`. PAS de JSON, PAS de Zustand persist. Clé volatile simple intentionnellement.
- **Erreurs silencieuses** : Le `try/catch` global avec `finally` est intentionnel — jamais de crash utilisateur sur l'initialisation. Le `finally` garantit le marquage completed même en cas d'erreur partielle.
- **Gemma model ID** : Vérifier l'ID exact `'gemma3-1b-mediapipe'` dans `mobile/src/` — chercher `llmModelsConfig.ts` et confirmer la clé avant d'écrire le code.

### Project Structure Notes

- **Nouveau fichier** : `mobile/src/contexts/identity/services/FirstLaunchInitializer.ts`
- **Fichiers à modifier** :
  - `mobile/src/infrastructure/di/tokens.ts` — ajouter `TOKENS.FirstLaunchInitializer` (et `TOKENS.NPUDetectionService` si pas déjà présent)
  - `mobile/src/infrastructure/di/container.ts` — `registerSingleton(TOKENS.FirstLaunchInitializer, FirstLaunchInitializer)`
  - `mobile/src/MainApp.tsx` (ou fichier équivalent racine) — résoudre l'initializer et appeler `run()` après auth
- **Nouveau composant UI** : `mobile/src/screens/FirstLaunchProgressScreen.tsx` (ou bottom sheet si pattern existant)
- **Cross-context import** : `NPUDetectionService` vient de `contexts/Normalization/` — import direct acceptable dans `container.ts`
- **`llmModelsConfig.ts`** : Localiser ce fichier dans `mobile/src/` pour confirmer l'ID `gemma3-1b-mediapipe` avant implementation

### Testing Standards

- **Pattern BDD mobile** : Voir `mobile/tests/acceptance/story-3-1-captures-list.test.ts`
- **Mocks requis** :
  - `AsyncStorage` → mock via `@react-native-async-storage/async-storage/jest/async-storage-mock`
  - `NPUDetectionService` → mock manuel ou jest.mock
  - `LLMModelService` → mock `downloadModel`, `setModelForTask`, `isModelDownloaded`
  - `TranscriptionEngineService` → mock `setSelectedEngineType`
  - `settingsStore` → `useSettingsStore.getState().setAutoTranscriptionEnabled` spy
- **6 cas unitaires minimum** (voir Tasks 5.1) — chaque cas isolé avec beforeEach qui reset AsyncStorage mock
- **Scénarios BDD** :
  1. Premier lancement Pixel 9 → native recognition activé + Gemma configuré
  2. Second lancement (flag déjà `'true'`) → `run()` no-op
  3. Premier lancement Samsung → aucun setting modifié, marqué completed

### References

- [Source: mobile/src/contexts/Normalization/services/NPUDetectionService.ts] — Service détection NPU + interface NPUInfo + PIXEL_TPU_MAP
- [Source: mobile/src/infrastructure/di/container.ts] — Pattern registerSingleton + exemples existants
- [Source: mobile/src/infrastructure/di/tokens.ts] — Token symbols DI
- [Source: mobile/src/MainApp.tsx] — Point d'intégration après auth (à localiser)
- [Source: mobile/src/contexts/identity/hooks/useSyncUserFeatures.ts] — Exemple pattern hook post-auth
- [Source: mobile/stores/settingsStore.ts] — `setAutoTranscriptionEnabled()` + pattern store
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-021] — DI Transient First (et exceptions justifiées)
- [Source: _bmad-output/implementation-artifacts/stories/epic-13/13-1-di-lifecycle-transient-first.md] — Précédent story ADR-021 compliance pour contexte

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

1. **API NPUInfo corrigée** : La story utilisait des noms de propriétés incorrects dans le pseudocode (`manufacturer === 'Google'` au lieu de `'google'`, `npu.model` au lieu de `npu.deviceModel`, `chipsetGeneration >= 4` au lieu du parsing regex de `npu.generation`). L'implémentation utilise l'API réelle de `NPUDetectionService`.

2. **settingsStore API** : La story référençait `setAutoTranscriptionEnabled()` mais la vraie méthode Zustand est `setAutoTranscription()`. Corrigé dans l'implémentation.

3. **Pattern spy Zustand** : Le pattern `jest.spyOn(useSettingsStore.getState(), 'setAutoTranscription')` est instable entre tests en raison de l'immuabilité Zustand. Solution adoptée : `useSettingsStore.setState({ autoTranscriptionEnabled: false })` dans `beforeEach` + vérification directe de `getState().autoTranscriptionEnabled` (pattern établi en story 24.3).

4. **Correction NPUDetectionService** : Correction d'une erreur TypeScript préexistante ligne 126 (`Platform.constants?.Model` → `(Platform.constants as Record<string, unknown>)?.['Model'] as string`). Cette erreur était silencieuse en runtime mais bloquait la compilation ts-jest des tests BDD.

5. **Download callback** : `run()` reçoit un `onProgress?` optionnel. Pour les tests BDD, le `when` step passe `jest.fn()` à `run()` pour que `downloadModel` reçoive un callback (nécessaire pour que `expect.any(Function)` matche).

6. **Pixel 10+ via génération** : `isPixel9Plus()` détecte les futurs modèles Pixel en parsant `npu.generation` (regex `/Tensor G(\d+)/i`, vérifiant `>= 4`), en plus de la liste explicite des modèles connus.

7. **[Code Review] Guard auth HF (C1)** : `gemma3-1b-mediapipe` a `requiresAuth: true` dans `llmModelsConfig.ts`. Sans guard préalable, `downloadModel()` throwait silencieusement ET `onProgress(0)` causait un flash UI à 0% avant l'échec. Fix : `canDownloadModel()` vérifié avant tout download ; si absent → skip propre sans callback ni flash. Nouveau test unitaire (Cas 7) + mock BDD mis à jour.

8. **[Code Review] Cleanup unmount hook (M2)** : `runFirstLaunch` étant module-level, `setIsVisible`/`setProgress` pouvaient être appelés sur un composant démonté. Fix : `isMountedRef` + setters guardés dans le hook.

9. **[Code Review] Taille Gemma depuis config (M1)** : `GEMMA_SIZE_MB = 555` hardcodé remplacé par `Math.round(MODEL_CONFIGS['gemma3-1b-mediapipe'].expectedSize / (1024 * 1024))` — source de vérité unique.

10. **[Code Review] Tests hook (H1)** : `useFirstLaunchInitialization` n'avait aucun test. Créé `__tests__/useFirstLaunchInitialization.test.ts` avec 6 cas via `renderHook` (@testing-library/react-native).

11. **[Code Review] testID + accessibilité (M3)** : `testID` ajoutés sur titre, barre de progression, texte de progression et bouton skip. `accessibilityRole` + `accessibilityLabel` ajoutés sur le bouton.

12. **[Code Review] Commentaire JSDoc + nommage test (M4)** : Commentaire sur `PIXEL_9_PLUS_MODELS` corrigé (Pixel 9a est dans la liste explicite ET couvert par le fallback génération). Test `isPixel9Plus` renommé en conséquence.

### File List

**Fichiers créés :**
- `pensieve/mobile/src/contexts/identity/services/FirstLaunchInitializer.ts`
- `pensieve/mobile/src/hooks/initialization/useFirstLaunchInitialization.ts`
- `pensieve/mobile/src/components/FirstLaunchProgress.tsx`
- `pensieve/mobile/src/contexts/identity/services/__tests__/FirstLaunchInitializer.test.ts`
- `pensieve/mobile/src/hooks/initialization/__tests__/useFirstLaunchInitialization.test.ts`
- `pensieve/mobile/tests/acceptance/features/story-24-4-first-launch.feature`
- `pensieve/mobile/tests/acceptance/story-24-4.test.ts`

**Fichiers modifiés :**
- `pensieve/mobile/src/infrastructure/di/tokens.ts` — ajout token `FirstLaunchInitializer`
- `pensieve/mobile/src/infrastructure/di/container.ts` — enregistrement `registerSingleton`
- `pensieve/mobile/src/components/MainApp.tsx` — intégration hook + composant
- `pensieve/mobile/src/contexts/Normalization/services/NPUDetectionService.ts` — correction erreur TS préexistante ligne 126
