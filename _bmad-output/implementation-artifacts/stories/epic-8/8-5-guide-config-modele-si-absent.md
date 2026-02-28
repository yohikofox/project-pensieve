# Story 8.5 : Guide Config Modèle LLM Si Absent

Status: done

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

En tant qu'**utilisateur ayant activé la digestion IA (post-processing LLM) mais sans modèle téléchargé**,
je veux **voir un guide clair m'indiquant qu'aucun modèle LLM n'est configuré et me proposant de le faire**,
afin de **comprendre pourquoi l'analyse de ma capture n'est pas disponible et pouvoir agir immédiatement**.

**GitHub Issue :** [#2 - feat: Guider l'utilisateur vers la configuration d'un modèle si aucun n'est défini](https://github.com/yohikofox/pensieve/issues/2)

## Contexte

### Ce qui est déjà implémenté (Story 2.7 — DONE)

Story 2.7 a implémenté le guide pour les **modèles Whisper** :
- `ModelConfigPrompt.tsx` : modal avant capture audio si aucun modèle Whisper
- `hasModelAvailable` dans `captureDetailStore` : état booléen de disponibilité Whisper
- Badge "Modèle requis" dans `CaptureHeader.tsx` et `AudioCaptureCard.tsx`
- `checkModelAvailability()` dans `useCaptureDetailInit.ts`

### Ce qui manque (Story 8.5 — à implémenter)

Pour les **modèles LLM de digestion** (PostProcessing / CaptureAnalysis) :
- `PostProcessingService.isEnabled()` retourne `false` silencieusement si aucun modèle
- `CaptureAnalysisService.doInitialize()` fait un `console.warn` uniquement si aucun modèle
- **Aucune UI** ne guide l'utilisateur vers `LLMSettingsScreen`
- `SettingsScreen` affiche "Désactivé" même quand LLM est activé mais sans modèle téléchargé

### Scénario problématique actuel

```
Utilisateur :
  1. Active "AI Enhancement" dans Settings → LLM activé mais aucun modèle téléchargé
  2. Enregistre une capture audio
  3. La transcription se fait (Whisper/Native OK)
  4. Ouvre le détail de la capture
  5. Voit la section "Analyse IA" → aucune analyse disponible
  6. 🤔 Ne sait pas POURQUOI il n'y a pas d'analyse ni COMMENT y remédier
```

### Décision architecturale

S'appuyer sur le **pattern établi par Story 2.7** :
- Nouveau state `hasLLMModelAvailable: boolean | null` dans `captureDetailStore` (parallèle à `hasModelAvailable`)
- Nouveau check `checkLLMModelAvailability()` dans `useCaptureDetailInit.ts`
- Banner/section "Configurer un modèle LLM" dans `AnalysisCard.tsx`
- Mise à jour `llmStatusLabel` dans `SettingsScreen` pour distinguer "Désactivé" vs "Aucun modèle"

## Acceptance Criteria

### AC1 : Détection disponibilité modèle LLM dans CaptureDetail

**Given** un utilisateur ouvre le détail d'une capture (audio ou texte)
**When** `useCaptureDetailInit` s'initialise
**Then** `hasLLMModelAvailable` est calculé via `ILLMModelService.getBestAvailableModel()`
**And** si aucun modèle LLM n'est téléchargé → `hasLLMModelAvailable = false`
**And** si au moins un modèle LLM est téléchargé → `hasLLMModelAvailable = true`
**And** si la vérification échoue → `hasLLMModelAvailable = null` (état inconnu, pas de guide affiché)
**And** ce check est indépendant de `hasModelAvailable` (Whisper)

### AC2 : Guide LLM dans AnalysisCard quand aucun modèle configuré

**Given** l'utilisateur ouvre le détail d'une capture
**And** `hasLLMModelAvailable === false`
**And** `isPostProcessingEnabled === true` (LLM est censé fonctionner)
**When** la section "Analyse IA" (`AnalysisCard`) s'affiche
**Then** un banner/section "Configurer un modèle LLM" est visible à l'intérieur de `AnalysisCard`
**And** le message explique : "Téléchargez un modèle LLM pour activer l'analyse automatique"
**And** un bouton "Configurer" navigue vers `LLMSettings`
**And** les boutons d'analyse individuels (résumé, idées, actions) ne sont PAS affichés (éviter confusion)

**Given** `hasLLMModelAvailable === true` OU `isPostProcessingEnabled === false`
**When** l'utilisateur ouvre la section Analyse IA
**Then** le guide LLM N'apparaît PAS
**And** le comportement normal de `AnalysisCard` est préservé

### AC3 : Navigation fonctionnelle vers LLMSettings

**Given** le guide LLM est visible dans `AnalysisCard`
**When** l'utilisateur appuie sur "Configurer"
**Then** il est navigué vers l'écran `LLMSettings` (dans le stack Settings)
**And** il peut télécharger ou sélectionner un modèle depuis cet écran
**And** quand il revient à CaptureDetail, le guide est toujours affiché (pas d'auto-refresh)

**Note :** La navigation `'LLMSettings' as never` suit le pattern existant (`CaptureHeader.tsx:277` pour WhisperSettings).

### AC4 : SettingsScreen — statut LLM plus informatif

**Given** l'utilisateur est sur l'écran Settings
**And** LLM post-processing est activé (`isPostProcessingEnabled === true`)
**And** aucun modèle n'est téléchargé (`hasDownloadedModels() === false`)
**When** il voit la ligne "AI Enhancement" dans la section Transcription
**Then** le label de statut affiche **"Aucun modèle"** (au lieu de "Désactivé")
**And** aucune icône d'alerte supplémentaire n'est nécessaire (label suffit)

**Given** LLM est activé ET au moins un modèle est téléchargé
**When** il voit la ligne "AI Enhancement"
**Then** le label de statut affiche le nom du modèle sélectionné OU "Activé" (comportement existant préservé)

**Given** LLM est désactivé (`isPostProcessingEnabled === false`)
**Then** le label affiche "Désactivé" (comportement existant préservé)

### AC5 : Tests unitaires et BDD

**Given** l'implémentation est en place
**When** les tests sont exécutés
**Then** les tests unitaires de `checkLLMModelAvailability()` couvrent les cas :
  - Modèle disponible (`getBestAvailableModel()` retourne un id) → `true`
  - Aucun modèle (`getBestAvailableModel()` retourne `null`) → `false`
  - Erreur de service → `null`
**And** les tests BDD (`story-8-5-guide-config-modele-si-absent.feature`) couvrent AC1, AC2, AC3
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression

## Tasks / Subtasks

### Task 1 : Ajouter `hasLLMModelAvailable` dans `captureDetailStore` (AC1)

- [x] Subtask 1.1 : Ouvrir `mobile/src/stores/captureDetailStore.ts`
- [x] Subtask 1.2 : Ajouter le state initial :
  ```typescript
  hasLLMModelAvailable: null as boolean | null,
  ```
- [x] Subtask 1.3 : Ajouter le setter dans les actions du store :
  ```typescript
  setHasLLMModelAvailable: (available: boolean | null) => set({ hasLLMModelAvailable: available }),
  ```
- [x] Subtask 1.4 : Ajouter `hasLLMModelAvailable` et `setHasLLMModelAvailable` à l'interface `CaptureDetailState`
- [x] Subtask 1.5 : Vérifier que `resetStore()` réinitialise `hasLLMModelAvailable` à `null`

### Task 2 : Ajouter `checkLLMModelAvailability()` dans `useCaptureDetailInit.ts` (AC1)

- [x] Subtask 2.1 : Ouvrir `mobile/src/hooks/useCaptureDetailInit.ts`
- [x] Subtask 2.2 : Ajouter le setter depuis le store :
  ```typescript
  const setStoreHasLLMModelAvailable = useCaptureDetailStore(
    (state) => state.setHasLLMModelAvailable,
  );
  ```
- [x] Subtask 2.3 : Implémenter la fonction de check (après `checkModelAvailability`) :
  ```typescript
  /**
   * Check if an LLM model is available for digestion/analysis
   */
  const checkLLMModelAvailability = useCallback(async () => {
    try {
      const llmModelService = container.resolve<ILLMModelService>(TOKENS.ILLMModelService);
      const bestModel = await llmModelService.getBestAvailableModel();
      setStoreHasLLMModelAvailable(bestModel !== null);
    } catch (err) {
      console.error(
        "[useCaptureDetailInit] Failed to check LLM model availability:",
        err,
      );
      setStoreHasLLMModelAvailable(null); // Unknown state — guide non affiché
    }
  }, [setStoreHasLLMModelAvailable]);
  ```
- [x] Subtask 2.4 : Ajouter l'Effect 5 (renommer l'existing Effect 5 → 6) :
  ```typescript
  // Effect 5: Check LLM model availability (for AnalysisCard guide)
  useEffect(() => {
    checkLLMModelAvailability();
  }, [checkLLMModelAvailability]);
  ```
- [x] Subtask 2.5 : Ajouter `hasLLMModelAvailable` au return de `useCaptureDetailInit` (si utilisé directement)
- [x] Subtask 2.6 : Vérifier les imports — `ILLMModelService` depuis `../../contexts/Normalization/domain/ILLMModelService`, `TOKENS` depuis `../../infrastructure/di/tokens`

### Task 3 : Ajouter le guide LLM dans `AnalysisCard.tsx` (AC2, AC3)

- [x] Subtask 3.1 : Ouvrir `mobile/src/components/capture/AnalysisCard.tsx`
- [x] Subtask 3.2 : Lire `hasLLMModelAvailable` et `isPostProcessingEnabled` depuis les stores :
  ```typescript
  const hasLLMModelAvailable = useCaptureDetailStore((state) => state.hasLLMModelAvailable);
  const isLLMEnabled = useSettingsStore((state) => state.llm.isEnabled);
  ```
- [x] Subtask 3.3 : Importer `useNavigation` depuis React Navigation :
  ```typescript
  import { useNavigation } from '@react-navigation/native';
  ```
- [x] Subtask 3.4 : Ajouter le handler de navigation :
  ```typescript
  const navigation = useNavigation();
  const handleConfigureLLM = useCallback(() => {
    navigation.navigate('LLMSettings' as never);
  }, [navigation]);
  ```
- [x] Subtask 3.5 : Calculer la condition d'affichage du guide :
  ```typescript
  const showLLMGuide = hasLLMModelAvailable === false && isLLMEnabled;
  ```
- [x] Subtask 3.6 : Ajouter le composant guide **avant** les sections d'analyse (si `showLLMGuide`) :
  ```tsx
  {showLLMGuide && (
    <View style={[styles.llmGuideContainer, { backgroundColor: themeColors.cardBg, borderColor: themeColors.borderDefault }]}>
      <Feather name="cpu" size={24} color={colors.warning[500]} />
      <Text style={[styles.llmGuideTitle, { color: themeColors.textPrimary }]}>
        {t('analysisCard.llmGuide.title', 'Modèle LLM requis')}
      </Text>
      <Text style={[styles.llmGuideDescription, { color: themeColors.textSecondary }]}>
        {t('analysisCard.llmGuide.description', 'Téléchargez un modèle LLM pour activer l\'analyse automatique de vos captures.')}
      </Text>
      <TouchableOpacity
        style={[styles.llmGuideButton, { backgroundColor: themeColors.primaryButton ?? colors.primary[500] }]}
        onPress={handleConfigureLLM}
        activeOpacity={0.8}
      >
        <Text style={styles.llmGuideButtonText}>
          {t('analysisCard.llmGuide.configure', 'Configurer un modèle')}
        </Text>
      </TouchableOpacity>
    </View>
  )}
  ```
- [x] Subtask 3.7 : Ajouter les styles correspondants dans `StyleSheet.create()`
- [x] Subtask 3.8 : Conditionner l'affichage des boutons d'analyse — les masquer quand `showLLMGuide` est `true` :
  ```tsx
  {!showLLMGuide && (
    // ... boutons handleGenerateAnalysis, handleAnalyzeAll existants
  )}
  ```

### Task 4 : Mise à jour `SettingsScreen.tsx` — label LLM (AC4)

- [x] Subtask 4.1 : Ouvrir `mobile/src/screens/settings/SettingsScreen.tsx`
- [x] Subtask 4.2 : Localiser la logique qui initialise `llmStatusLabel` (dans un `useEffect` ou équivalent)
- [x] Subtask 4.3 : Mettre à jour la logique pour distinguer "Aucun modèle" vs "Désactivé" :
  ```typescript
  // Logique mise à jour pour llmStatusLabel
  // Lire isPostProcessingEnabled ET hasDownloadedModels()
  const llmEnabled = await llmModelService.isPostProcessingEnabled();
  const hasModels = hasDownloadedModels(); // function from llmSettingsScreenStore

  if (!llmEnabled) {
    setLlmStatusLabel(t('common.disabled', 'Désactivé'));
  } else if (!hasModels) {
    setLlmStatusLabel(t('settings.llm.noModel', 'Aucun modèle'));
  } else {
    const selectedModel = await llmModelService.getSelectedModel();
    setLlmStatusLabel(selectedModel
      ? llmModelService.getModelConfig(selectedModel).name
      : t('common.enabled', 'Activé'));
  }
  ```
- [x] Subtask 4.4 : Importer les dépendances nécessaires si pas déjà présentes (`hasDownloadedModels`, `ILLMModelService`, `TOKENS`)

### Task 5 : Tests BDD — Scénarios story 8.5 (AC5)

- [x] Subtask 5.1 : Créer `mobile/tests/acceptance/features/story-8-5-guide-config-modele-si-absent.feature`
- [x] Subtask 5.2 : Écrire les scénarios Gherkin (voir section "Scénarios BDD" ci-dessous)
- [x] Subtask 5.3 : Créer `mobile/tests/acceptance/story-8-5-guide-config-modele-si-absent.test.ts` avec les step definitions
- [x] Subtask 5.4 : Mocker `ILLMModelService` via `TOKENS.ILLMModelService` dans le container TSyringe

### Task 6 : Tests unitaires — `checkLLMModelAvailability` (AC5)

- [x] Subtask 6.1 : Ouvrir ou créer `mobile/src/hooks/__tests__/useCaptureDetailInit.test.ts`
- [x] Subtask 6.2 : Ajouter les tests pour `checkLLMModelAvailability` :
  ```
  Cas 1 — getBestAvailableModel retourne un id → hasLLMModelAvailable = true
  Cas 2 — getBestAvailableModel retourne null → hasLLMModelAvailable = false
  Cas 3 — getBestAvailableModel lève une erreur → hasLLMModelAvailable = null
  ```
- [x] Subtask 6.3 : Pattern de mock `ILLMModelService` identique au mock existant pour `TranscriptionModelService`

### Task 7 : Validation finale (AC5)

- [x] Subtask 7.1 : `npm run test:unit` dans `pensieve/mobile/` — zéro régression
- [x] Subtask 7.2 : `npm run test:acceptance` dans `pensieve/mobile/` — zéro régression
- [x] Subtask 7.3 : `npm run test:architecture` — conformité ADR maintenue
- [x] Subtask 7.4 : Fermer l'issue GitHub #2 avec référence au commit

## Scénarios BDD (Feature File)

```gherkin
# language: fr
Fonctionnalité: Guide Configuration Modèle LLM Si Absent
  En tant qu'utilisateur ayant activé la digestion IA mais sans modèle LLM téléchargé
  Je veux être guidé vers la configuration du modèle LLM
  Afin de comprendre pourquoi l'analyse n'est pas disponible et pouvoir y remédier

  Contexte:
    Étant donné que je suis un utilisateur authentifié
    Et que l'application est lancée
    Et que la digestion IA (post-processing) est activée dans les settings

  Scénario: Guide visible quand aucun modèle LLM n'est téléchargé
    Étant donné qu'aucun modèle LLM n'est téléchargé
    Et qu'une capture audio existe avec état "ready"
    Quand j'ouvre le détail de cette capture
    Alors je vois le banner "Modèle LLM requis" dans la section Analyse IA
    Et je vois le message "Téléchargez un modèle LLM pour activer l'analyse automatique"
    Et je vois le bouton "Configurer un modèle"
    Et les boutons d'analyse individuels ne sont PAS affichés

  Scénario: Navigation vers LLMSettings depuis le guide
    Étant donné qu'aucun modèle LLM n'est téléchargé
    Et que je vois le banner "Modèle LLM requis"
    Quand j'appuie sur "Configurer un modèle"
    Alors je suis navigué vers l'écran LLMSettings
    Et je peux voir la liste des modèles LLM disponibles

  Scénario: Guide absent quand modèle LLM disponible
    Étant donné qu'un modèle LLM est téléchargé
    Et qu'une capture audio existe avec état "ready"
    Quand j'ouvre le détail de cette capture
    Alors le banner "Modèle LLM requis" n'est PAS visible
    Et les boutons d'analyse normaux sont affichés

  Scénario: Guide absent quand digestion IA est désactivée
    Étant donné qu'aucun modèle LLM n'est téléchargé
    Et que la digestion IA est désactivée dans les settings
    Et qu'une capture audio existe avec état "ready"
    Quand j'ouvre le détail de cette capture
    Alors le banner "Modèle LLM requis" n'est PAS visible

  Scénario: hasLLMModelAvailable null en cas d'erreur de service
    Étant donné que ILLMModelService.getBestAvailableModel lève une erreur
    Et qu'une capture audio existe
    Quand j'ouvre le détail de cette capture
    Alors aucun banner LLM n'est affiché (état inconnu ignoré)
    Et aucune erreur visible à l'utilisateur
```

## Dev Notes

### 🔑 Fichiers Principaux

**À modifier :**
```
pensieve/mobile/src/stores/captureDetailStore.ts            ← Task 1 : state + setter
pensieve/mobile/src/hooks/useCaptureDetailInit.ts           ← Task 2 : checkLLMModelAvailability
pensieve/mobile/src/components/capture/AnalysisCard.tsx     ← Task 3 : banner LLM guide
pensieve/mobile/src/screens/settings/SettingsScreen.tsx     ← Task 4 : llmStatusLabel
```

**À créer :**
```
pensieve/mobile/tests/acceptance/features/story-8-5-guide-config-modele-si-absent.feature
pensieve/mobile/tests/acceptance/story-8-5-guide-config-modele-si-absent.test.ts
```

**À ne PAS modifier :**
```
pensieve/mobile/src/components/modals/ModelConfigPrompt.tsx    ← Story 2.7, ne pas toucher
pensieve/mobile/src/contexts/Normalization/services/PostProcessingService.ts  ← pas de logique UI
pensieve/mobile/src/contexts/Normalization/services/CaptureAnalysisService.ts ← pas de logique UI
```

### 🏗️ Pattern de Référence — Story 2.7

**Parallèle exact avec Whisper :**

| Story 2.7 (Whisper) | Story 8.5 (LLM) |
|---|---|
| `hasModelAvailable` dans store | `hasLLMModelAvailable` dans store |
| `checkModelAvailability()` dans `useCaptureDetailInit` | `checkLLMModelAvailability()` dans `useCaptureDetailInit` |
| `TranscriptionModelService.getBestAvailableModel()` | `ILLMModelService.getBestAvailableModel()` |
| Badge dans `CaptureHeader.tsx` | Banner dans `AnalysisCard.tsx` |
| Navigate to `WhisperSettings` | Navigate to `LLMSettings` |

### 🔗 Token DI pour ILLMModelService

```typescript
// tokens.ts — vérifier le token exact
TOKENS.ILLMModelService  // ou chercher dans src/infrastructure/di/tokens.ts
```

Utilisation dans le hook :
```typescript
const llmModelService = container.resolve<ILLMModelService>(TOKENS.ILLMModelService);
```

### 📋 Navigation vers LLMSettings

Pattern existant dans `CaptureHeader.tsx:277` :
```typescript
navigation.navigate("WhisperSettings" as never);
```

Pour LLM :
```typescript
navigation.navigate("LLMSettings" as never);
```

`LLMSettings` est déclaré dans `SettingsNavigationTypes.ts:12` et routé dans `SettingsStackNavigator.tsx:69-70`.

### 🎨 Design : Banner LLM Guide dans AnalysisCard

Le banner doit être **minimal et cohérent** avec le style existant de `AnalysisCard`. Ne pas créer un composant séparé (YAGNI) — inline dans `AnalysisCard` suffit.

Pattern visuel : similaire au badge "Modèle requis" existant dans `CaptureHeader.tsx:270-277` (warning orange, icône, texte, bouton).

### 📋 `hasDownloadedModels()` dans SettingsScreen

Cette fonction est importée depuis `llmSettingsScreenStore` :
```typescript
import { useLLMSettingsScreenStore, hasDownloadedModels } from '../../stores/llmSettingsScreenStore';
```

Elle vérifie si des modèles LLM sont téléchargés dans les listes `tpuModels` et `standardModels` du store.
**Attention :** Cette fonction lit depuis le store UI de LLMSettingsScreen qui n'est initialisé que quand `LLMSettingsScreen` est monté. Pour `SettingsScreen`, utiliser directement `ILLMModelService.getBestAvailableModel()` est plus fiable.

### ⚠️ Garde-Fous Critiques

1. **Ne pas ajouter de logique UI dans PostProcessingService/CaptureAnalysisService** — la séparation des responsabilités doit être maintenue
2. **Ne pas modifier ModelConfigPrompt.tsx** — spécifique Whisper, story 2.7
3. **`hasLLMModelAvailable === null` → ne pas afficher le guide** — état inconnu ne doit pas perturber l'utilisateur
4. **Le guide n'est affiché que si LLM est ACTIVÉ mais sans modèle** — si LLM est désactivé par l'utilisateur, pas de guide (respects de la décision utilisateur)
5. **Pas de navigation automatique** — l'utilisateur doit appuyer explicitement sur "Configurer"

### 🏗️ Conformité ADR

- **ADR-021 (DI Lifecycle)** : Résolution lazy via `container.resolve()` dans `useCallback` — conforme pattern établi dans `useCaptureDetailInit.ts`
- **ADR-022 (AsyncStorage)** : Aucune donnée persistée pour cette story
- **ADR-023 (Result Pattern)** : `getBestAvailableModel()` retourne `LLMModelId | null` (pas de `Result<T>` nécessaire — valeur nulle suffit pour ce check de disponibilité)
- **ADR-024 (Clean Code)** : Constantes nommées (`showLLMGuide`), Single Responsibility (vérification dans hook, UI dans composant)

### Project Structure Notes

```
pensieve/mobile/src/
├── stores/
│   └── captureDetailStore.ts          ← MODIFIER : state hasLLMModelAvailable
├── hooks/
│   ├── useCaptureDetailInit.ts        ← MODIFIER : checkLLMModelAvailability
│   └── __tests__/
│       └── useCaptureDetailInit.test.ts ← MODIFIER : nouveaux tests
├── components/capture/
│   └── AnalysisCard.tsx               ← MODIFIER : banner LLM guide
└── screens/settings/
    └── SettingsScreen.tsx             ← MODIFIER : llmStatusLabel logic

tests/acceptance/
├── features/
│   └── story-8-5-guide-config-modele-si-absent.feature ← CRÉER
└── story-8-5-guide-config-modele-si-absent.test.ts      ← CRÉER
```

### References

- [Source: GitHub Issue #2](https://github.com/yohikofox/pensieve/issues/2) — Spécification originale
- [Source: `pensieve/mobile/src/components/modals/ModelConfigPrompt.tsx`] — Pattern référence (Whisper)
- [Source: `pensieve/mobile/src/hooks/useCaptureDetailInit.ts#157-184`] — Pattern `checkModelAvailability()` à dupliquer pour LLM
- [Source: `pensieve/mobile/src/stores/captureDetailStore.ts#56`] — `hasModelAvailable` à dupliquer pour LLM
- [Source: `pensieve/mobile/src/components/capture/AnalysisCard.tsx`] — Composant à modifier
- [Source: `pensieve/mobile/src/navigation/SettingsNavigationTypes.ts#12`] — Route `LLMSettings`
- [Source: `pensieve/mobile/src/screens/settings/SettingsScreen.tsx#50`] — `llmStatusLabel` à améliorer
- [Source: `_bmad-output/implementation-artifacts/stories/epic-8/8-4-transcription-native-par-defaut.md`] — Story précédente (contexte AsyncStorage patterns)
- [Source: ADR-021] — DI Lifecycle (Transient First)
- [Source: ADR-023] — Result Pattern

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

- Implémentation complète suivant le pattern Story 2.7 (Whisper) pour LLM.
- `hasLLMModelAvailable` ajouté dans `captureDetailStore` avec valeurs `true/false/null`.
- `checkLLMModelAvailability()` dans `useCaptureDetailInit` — résolution lazy de `ILLMModelService` via `container.resolve()` dans `useCallback` (conforme ADR-021).
- Banner LLM guide dans `AnalysisCard.tsx` avec condition `showLLMGuide = hasLLMModelAvailable === false && isLLMEnabled`. Boutons d'analyse masqués quand guide affiché.
- `SettingsScreen.tsx` : cas "Aucun modèle" ajouté quand LLM activé mais `getBestAvailableModel()` retourne null.
- 5 scénarios BDD passent (100%). 4 tests unitaires `checkLLMModelAvailability` passent.
- Aucune régression introduite — les 11 échecs unit et 6 échecs architecture préexistaient (APIs stale + ADR-031 violations hors scope).

### File List

**Fichiers modifiés :**
- `pensieve/mobile/src/stores/captureDetailStore.ts`
- `pensieve/mobile/src/hooks/useCaptureDetailInit.ts`
- `pensieve/mobile/src/components/capture/AnalysisCard.tsx`
- `pensieve/mobile/src/screens/settings/SettingsScreen.tsx`
- `pensieve/mobile/src/hooks/__tests__/useCaptureDetailInit.test.ts`

**Fichiers créés :**
- `pensieve/mobile/tests/acceptance/features/story-8-5-guide-config-modele-si-absent.feature`
- `pensieve/mobile/tests/acceptance/story-8-5-guide-config-modele-si-absent.test.ts`

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #2 — analyse approfondie du code existant (story 2.7 pattern, captureDetailStore, useCaptureDetailInit, AnalysisCard, SettingsScreen), stratégie parallèle Whisper→LLM définie | yohikofox |
| 2026-02-28 | Implémentation complète — Tasks 1-7 réalisées, 5/5 BDD passent, 4/4 tests unitaires passent, aucune régression | yohikofox |
