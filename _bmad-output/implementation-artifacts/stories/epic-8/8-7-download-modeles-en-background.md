# Story 8.7 : Téléchargement de Modèles en Arrière-Plan avec Notification de Fin

Status: review

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

En tant qu'**utilisateur souhaitant télécharger un modèle LLM ou Whisper**,
je veux **que le téléchargement continue même quand l'application passe en arrière-plan**,
afin de **pouvoir utiliser d'autres applications pendant ce temps et être notifié par une notification push quand c'est terminé (succès ou échec)**.

**GitHub Issue :** [#18 - feat: Téléchargement de modèles en arrière-plan avec notification de fin](https://github.com/yohikofox/pensieve/issues/18)

## Contexte

### Ce qui est déjà implémenté

#### `LLMModelService.ts` — React Native Background Downloader (833 lignes)

`LLMModelService.ts` utilise **déjà** `@kesha-antonov/react-native-background-downloader` (v4.4.5) pour les modèles LLM :

```typescript
const task = createDownloadTask({
  id: modelId,
  url: downloadUrl,
  destination: modelFile.uri,
  headers: authHeaders
});

task.begin(...)
  .progress(({ bytesDownloaded, bytesTotal }) => {...})
  .done(({ location }) => {...})
  .error(({ error }) => {...});
```

Les fonctionnalités de pause/resume/cancel, la récupération après crash (`recoverInterruptedDownloads()`), et le backoff exponentiel (`downloadModelWithRetry()`) sont déjà implémentés.

**Ce qui manque :** Le téléchargement s'interrompt quand l'app passe en arrière-plan, et aucune notification n'est envoyée à la fin.

#### `TranscriptionModelService.ts` — expo/fetch streaming (503 lignes)

`TranscriptionModelService.ts` utilise **`expo/fetch` avec ReadableStream** pour les modèles Whisper :

```typescript
const response = await fetch(url, { headers: authHeaders });
const reader = response.body!.getReader();
// ... streaming manuel vers fichier
```

**Ce qui manque :** Cette approche est 100% foreground — le téléchargement s'arrête dès que l'app passe en arrière-plan. Les modèles Whisper (75Mo à 3.1Go) nécessitent impérativement le background.

#### Dépendances déjà installées

```json
"@kesha-antonov/react-native-background-downloader": "^4.4.5",
"expo-background-fetch": "^14.0.9",
"expo-task-manager": "^14.0.9",
"expo-notifications": "~0.32.16"
```

Toutes les bibliothèques nécessaires sont **déjà dans les dépendances** — pas d'installation requise.

#### Stores et UI existants

- `settingsStore.ts` — état LLM + Whisper persisté dans AsyncStorage
- `llmSettingsScreenStore.ts` — état UI-spécifique LLM screen
- `LLMSettingsScreen.tsx` + `LLMModelCard.tsx` — UI avec boutons Download/Pause/Resume/Cancel + progress bar
- `WhisperSettingsScreen.tsx` + `WhisperModelCard.tsx` — UI identique pour Whisper
- `useLLMDownloadRecovery.ts` — hook de récupération au démarrage

### Ce qui manque (Story 8.7 — à implémenter)

1. **Téléchargement LLM en background** : configurer `@kesha-antonov/react-native-background-downloader` pour le mode background (Android foreground service + iOS URLSession background)
2. **Téléchargement Whisper en background** : migrer `TranscriptionModelService` de `expo/fetch` streaming vers `react-native-background-downloader`
3. **Notification de fin** : succès et échec via `expo-notifications`
4. **Notification persistante Android** : pendant le téléchargement (foreground service)
5. **Synchronisation état au retour** : rafraîchir UI quand l'utilisateur revient dans l'app
6. **Paramètre WiFi uniquement** (optionnel selon complexité — voir AC8)

### Comportement actuel (problème)

```
Utilisateur :
  1. Lance le téléchargement d'un modèle LLM (ex: Qwen2.5 3B = ~2Go)
  2. Quitte l'app pour répondre à un message
  3. 🔴 Le téléchargement s'interrompt / met en pause
  4. L'utilisateur doit rouvrir l'app et souvent relancer manuellement
  5. 🔴 Aucune notification quand c'est terminé
```

### Comportement attendu après Story 8.7

```
Utilisateur :
  1. Lance le téléchargement d'un modèle LLM
  2. Quitte l'app pour répondre à un message
  3. ✅ Le téléchargement continue en arrière-plan
  4. ✅ Android : notification persistante "Téléchargement en cours..."
  5. L'utilisateur reçoit une notification push "Modèle téléchargé avec succès"
  6. ✅ Tap sur la notification → ouvre l'app sur l'écran Settings/LLM
  7. Le modèle est prêt à l'emploi
```

## Acceptance Criteria

### AC1 : Téléchargement LLM en arrière-plan

**Given** je lance le téléchargement d'un modèle LLM depuis `LLMSettingsScreen`
**When** le téléchargement est en cours et je quitte l'application
**Then** le téléchargement continue en arrière-plan (Android natif + iOS URLSession background)
**And** le pourcentage de progression est préservé (pas de restart depuis 0)
**And** le téléchargement persiste même si l'écran se verrouille

**Given** le téléchargement se termine en arrière-plan
**When** je reviens dans l'application
**Then** le modèle est marqué comme "téléchargé" dans l'UI
**And** je peux le sélectionner immédiatement sans action supplémentaire

### AC2 : Téléchargement Whisper en arrière-plan

**Given** je lance le téléchargement d'un modèle Whisper depuis `WhisperSettingsScreen`
**When** le téléchargement est en cours et je quitte l'application
**Then** le téléchargement continue en arrière-plan (même comportement que AC1)
**And** la vérification SHA256 (pour tiny/base/small) s'effectue à la fin

**Given** le téléchargement Whisper se termine en arrière-plan
**When** je reviens dans l'application
**Then** le modèle est disponible et peut être sélectionné

### AC3 : Notification de fin — succès

**Given** un téléchargement de modèle (LLM ou Whisper) se termine avec succès
**When** la completion event est reçue
**Then** une notification push locale est envoyée :
  - Titre : "Modèle téléchargé"
  - Corps : "[Nom du modèle] est prêt à l'emploi"
**And** la notification s'affiche même si l'app est en arrière-plan ou fermée (foreground)
**And** un tap sur la notification ouvre l'app sur l'écran approprié (LLM Settings ou Whisper Settings)
**And** la notification est envoyée uniquement si les permissions ont été accordées

### AC4 : Notification de fin — échec

**Given** un téléchargement de modèle échoue (réseau, espace disque, etc.)
**When** l'erreur est reçue
**Then** une notification push locale est envoyée :
  - Titre : "Échec du téléchargement"
  - Corps : "[Nom du modèle] — Appuyer pour réessayer"
**And** un tap sur la notification ouvre l'app sur l'écran approprié
**And** l'utilisateur peut relancer le téléchargement depuis l'UI

### AC5 : Notification persistante Android pendant téléchargement

**Given** je lance un téléchargement sur Android
**When** le téléchargement démarre
**Then** une notification persistante s'affiche dans le panneau de notifications
**And** la notification montre le nom du modèle et le pourcentage de progression
**And** la notification est mise à jour à chaque événement de progression significatif (tous les ~10%)
**And** la notification disparaît automatiquement quand le téléchargement se termine ou est annulé

### AC6 : Demande de permissions notifications

**Given** je lance mon premier téléchargement depuis la Story 8.7
**When** le téléchargement démarre pour la première fois
**Then** le système demande les permissions de notification si elles ne sont pas encore accordées
**And** si l'utilisateur refuse → le téléchargement continue quand même (notifications optionnelles)
**And** si l'utilisateur accepte → les notifications fonctionnent pour tous les futurs téléchargements

### AC7 : Synchronisation de l'état au retour dans l'app

**Given** un téléchargement s'est terminé en arrière-plan
**When** je reviens dans l'application et navigue vers LLMSettingsScreen ou WhisperSettingsScreen
**Then** l'état de l'UI est synchronisé avec l'état réel du fichier sur disque
**And** les modèles téléchargés sont bien marqués "Downloaded"
**And** les modèles en cours de téléchargement affichent la progression correcte

### AC8 : Gestion des téléchargements multiples simultanés

**Given** plusieurs téléchargements sont en cours (ex: un LLM + un Whisper)
**When** ils se terminent (avec succès ou échec)
**Then** chaque modèle reçoit sa propre notification distincte
**And** les états sont gérés indépendamment (le succès d'un n'affecte pas l'autre)

### AC9 : Tests unitaires et BDD

**Given** l'implémentation est complète
**When** les tests sont exécutés
**Then** les scénarios BDD dans `story-8-7-download-modeles-en-background.feature` couvrent :
  - AC1/AC2 : Download continue en background (mock task completion while app backgrounded)
  - AC3 : Notification succès envoyée
  - AC4 : Notification échec envoyée
  - AC6 : Permission demandée au premier download
**And** les tests unitaires du `ModelDownloadNotificationService` couvrent :
  - Envoi notification succès avec payload correct
  - Envoi notification échec avec payload correct
  - Mise à jour notification persistante Android
  - Tap notification → navigate vers bon écran
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression

## Tasks / Subtasks

### Task 1 : Créer `ModelDownloadNotificationService` (AC3, AC4, AC5, AC6)

- [ ] Subtask 1.1 : Créer `mobile/src/contexts/Normalization/services/ModelDownloadNotificationService.ts`
- [ ] Subtask 1.2 : Définir l'interface `IModelDownloadNotificationService` dans `mobile/src/contexts/Normalization/domain/IModelDownloadNotificationService.ts` :
  ```typescript
  export interface IModelDownloadNotificationService {
    initialize(): Promise<void>;
    requestPermissions(): Promise<boolean>;
    notifyDownloadSuccess(modelId: string, modelName: string, screen: 'llm' | 'whisper'): Promise<void>;
    notifyDownloadError(modelId: string, modelName: string, screen: 'llm' | 'whisper'): Promise<void>;
    updateProgressNotification(modelId: string, modelName: string, progress: number): Promise<void>;
    dismissProgressNotification(modelId: string): Promise<void>;
  }
  ```
- [ ] Subtask 1.3 : Implémenter `initialize()` — appeler `expo-notifications` pour configurer le canal Android (channel ID `model-downloads`) :
  ```typescript
  await Notifications.setNotificationChannelAsync('model-downloads', {
    name: 'Téléchargements de modèles',
    importance: Notifications.AndroidImportance.DEFAULT,
    showBadge: false,
  });
  ```
- [ ] Subtask 1.4 : Implémenter `requestPermissions()` :
  ```typescript
  const { status } = await Notifications.requestPermissionsAsync();
  return status === 'granted';
  ```
- [ ] Subtask 1.5 : Implémenter `notifyDownloadSuccess(modelId, modelName, screen)` :
  - `schedulePushNotificationAsync` avec trigger null (immediate)
  - Payload navigation : `{ screen: screen }` dans `data`
  - Titre : `"Modèle téléchargé"`, corps : `"${modelName} est prêt à l'emploi"`
- [ ] Subtask 1.6 : Implémenter `notifyDownloadError(modelId, modelName, screen)` :
  - Titre : `"Échec du téléchargement"`, corps : `"${modelName} — Appuyer pour réessayer"`
  - Payload navigation : `{ screen: screen, retry: modelId }`
- [ ] Subtask 1.7 : Implémenter `updateProgressNotification(modelId, modelName, progress)` — Android uniquement :
  - Utiliser `presentNotificationAsync` avec notification persistante (sticky)
  - Progressbar via Android notification `progress` options
  - Debounce à 10% d'intervalle minimum pour éviter le spam
- [ ] Subtask 1.8 : Implémenter `dismissProgressNotification(modelId)` :
  - `dismissNotificationAsync(notificationId)` pour la notification persistante
- [ ] Subtask 1.9 : Enregistrer dans container TSyringe (DI) et ajouter token dans `TOKENS`

### Task 2 : Intégrer les notifications dans `LLMModelService` (AC1, AC3, AC4, AC5)

- [ ] Subtask 2.1 : Injecter `IModelDownloadNotificationService` dans le constructeur de `LLMModelService`
- [ ] Subtask 2.2 : Dans `initialize()` — appeler `notificationService.initialize()` puis `notificationService.requestPermissions()` (une seule fois, persisté via AsyncStorage flag `@pensieve/notifications_requested`)
- [ ] Subtask 2.3 : Dans le handler `.done()` de `downloadModel()` :
  ```typescript
  .done(async ({ location }) => {
    await notificationService.dismissProgressNotification(modelId);
    await notificationService.notifyDownloadSuccess(modelId, modelName, 'llm');
    // ... existing logic
  })
  ```
- [ ] Subtask 2.4 : Dans le handler `.error()` :
  ```typescript
  .error(async ({ error }) => {
    await notificationService.dismissProgressNotification(modelId);
    await notificationService.notifyDownloadError(modelId, modelName, 'llm');
    // ... existing error logic
  })
  ```
- [ ] Subtask 2.5 : Dans le handler `.progress()` — appeler `updateProgressNotification()` :
  ```typescript
  .progress(async ({ bytesDownloaded, bytesTotal }) => {
    const progress = bytesDownloaded / bytesTotal;
    await notificationService.updateProgressNotification(modelId, modelName, progress);
    onProgress?.(progress);
  })
  ```
- [ ] Subtask 2.6 : Vérifier que `@kesha-antonov/react-native-background-downloader` v4.4.5 est configuré pour le background :
  - Consulter l'API v4.x pour les options `isAllowedOverMetered`, `allowedOverRoaming`
  - Sur Android : vérifier si la librairie crée automatiquement un foreground service ou si configuration manuelle requise
  - Sur iOS : vérifier si la configuration URLSession background est automatique dans v4.x

### Task 3 : Migrer `TranscriptionModelService` vers background downloader (AC2)

- [ ] Subtask 3.1 : Ouvrir `mobile/src/contexts/Normalization/services/TranscriptionModelService.ts`
- [ ] Subtask 3.2 : Remplacer la méthode `downloadModel()` qui utilise `expo/fetch` streaming par `@kesha-antonov/react-native-background-downloader` :
  ```typescript
  // AVANT (foreground only):
  const response = await fetch(url, { headers });
  const reader = response.body!.getReader();
  // ... streaming manuel

  // APRÈS (background compatible):
  const task = createDownloadTask({
    id: `whisper-${modelSize}`,
    url: downloadUrl,
    destination: modelFilePath,
    headers: {},
  });

  await new Promise<void>((resolve, reject) => {
    task
      .begin((expectedBytes) => { /* log expected */ })
      .progress(({ bytesDownloaded, bytesTotal }) => {
        const progress = bytesDownloaded / bytesTotal;
        notificationService.updateProgressNotification(`whisper-${modelSize}`, modelName, progress);
        onProgress?.(progress);
      })
      .done(async ({ location }) => {
        await notificationService.dismissProgressNotification(`whisper-${modelSize}`);
        // SHA256 check si disponible
        if (config.sha256) {
          const isValid = await this.verifyChecksum(location, config.sha256);
          if (!isValid) {
            await FileSystem.deleteAsync(location);
            notificationService.notifyDownloadError(`whisper-${modelSize}`, modelName, 'whisper');
            reject(new Error('Checksum mismatch'));
            return;
          }
        }
        await notificationService.notifyDownloadSuccess(`whisper-${modelSize}`, modelName, 'whisper');
        resolve();
      })
      .error(async ({ error }) => {
        await notificationService.dismissProgressNotification(`whisper-${modelSize}`);
        await notificationService.notifyDownloadError(`whisper-${modelSize}`, modelName, 'whisper');
        reject(error);
      });
    task.start();
  });
  ```
- [ ] Subtask 3.3 : Implémenter pause/resume/cancel pour les modèles Whisper (aligner avec les méthodes existantes de `LLMModelService`) :
  - Ajouter `this.activeWhisperTask` pour reference au task courant
  - Exposer `pauseDownload()`, `resumeDownload()`, `cancelDownload()` sur l'interface `ITranscriptionModelService`
- [ ] Subtask 3.4 : Implémenter récupération après crash pour Whisper (aligner avec `recoverInterruptedDownloads()` de LLM) :
  - Sauvegarder `@pensieve/whisper_download_resume_${modelSize}` dans AsyncStorage
  - Vérifier à l'initialisation si un téléchargement Whisper était en cours
- [ ] Subtask 3.5 : Injecter `IModelDownloadNotificationService` dans `TranscriptionModelService`
- [ ] Subtask 3.6 : Appeler `notificationService.requestPermissions()` au premier téléchargement (même flag que Task 2.2)

### Task 4 : Configurer les permissions iOS background mode (AC1, AC2)

- [ ] Subtask 4.1 : Vérifier `mobile/app.json` — ajouter le background mode si absent :
  ```json
  {
    "expo": {
      "ios": {
        "infoPlist": {
          "UIBackgroundModes": ["fetch", "background-processing"]
        }
      }
    }
  }
  ```
- [ ] Subtask 4.2 : Si `expo-task-manager` est requis pour iOS background fetch — créer le task :
  ```typescript
  // Dans un fichier dédié, e.g. backgroundTasks.ts
  import * as TaskManager from 'expo-task-manager';
  import * as BackgroundFetch from 'expo-background-fetch';

  const BACKGROUND_DOWNLOAD_TASK = 'BACKGROUND_MODEL_DOWNLOAD';

  TaskManager.defineTask(BACKGROUND_DOWNLOAD_TASK, async () => {
    // @kesha-antonov/react-native-background-downloader gère nativement
    // Ce task est un fallback iOS pour réveiller l'app
    return BackgroundFetch.BackgroundFetchResult.NewData;
  });
  ```
  **Note**: Si `react-native-background-downloader` v4.x gère iOS nativement via NSURLSession, ce task peut être optionnel — vérifier la documentation.
- [ ] Subtask 4.3 : Vérifier si `expo-notifications` nécessite configuration supplémentaire dans `app.json` pour iOS

### Task 5 : Gérer la navigation depuis les notifications (AC3, AC4)

- [ ] Subtask 5.1 : Configurer un `NotificationResponseListener` dans `App.tsx` (ou le provider global) :
  ```typescript
  useEffect(() => {
    const subscription = Notifications.addNotificationResponseReceivedListener((response) => {
      const { screen, retry } = response.notification.request.content.data;
      if (screen === 'llm') {
        // Navigate to LLM Settings screen
        navigationRef.current?.navigate('Settings', { screen: 'LLMSettings' });
      } else if (screen === 'whisper') {
        navigationRef.current?.navigate('Settings', { screen: 'WhisperSettings' });
      }
    });
    return () => subscription.remove();
  }, []);
  ```
- [ ] Subtask 5.2 : Vérifier la structure de navigation existante dans le projet pour utiliser le bon chemin de navigation (ne pas inventer de routes)

### Task 6 : Synchronisation état UI au retour dans l'app (AC7)

- [ ] Subtask 6.1 : Dans `LLMSettingsScreen`, ajouter un `useEffect` sur `AppState.addEventListener('change', state)` pour rafraîchir la liste des modèles quand l'app revient au premier plan :
  ```typescript
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (nextAppState) => {
      if (nextAppState === 'active') {
        refreshModels(); // Re-fetch download status from LLMModelService
      }
    });
    return () => subscription.remove();
  }, []);
  ```
- [ ] Subtask 6.2 : Même logique dans `WhisperSettingsScreen`
- [ ] Subtask 6.3 : Vérifier que `refreshModels()` appelle bien `modelService.isModelDownloaded()` pour chaque modèle (rafraîchissement filesystem, pas AsyncStorage cache)

### Task 7 : Tests BDD (AC9)

- [ ] Subtask 7.1 : Créer `mobile/tests/acceptance/features/story-8-7-download-modeles-en-background.feature`
- [ ] Subtask 7.2 : Écrire les scénarios Gherkin (voir section "Scénarios BDD" ci-dessous)
- [ ] Subtask 7.3 : Créer `mobile/tests/acceptance/story-8-7-download-modeles-en-background.test.ts` avec step definitions
- [ ] Subtask 7.4 : Mocker `ModelDownloadNotificationService`, `LLMModelService`, `TranscriptionModelService` via TSyringe container

### Task 8 : Tests unitaires — `ModelDownloadNotificationService` (AC9)

- [ ] Subtask 8.1 : Créer `mobile/src/contexts/Normalization/services/__tests__/ModelDownloadNotificationService.test.ts`
- [ ] Subtask 8.2 : Cas testés :
  ```
  Cas 1 — notifyDownloadSuccess → schedulePushNotificationAsync appelé avec payload correct
  Cas 2 — notifyDownloadError → notification avec titre "Échec" + data retry
  Cas 3 — updateProgressNotification → presentNotificationAsync appelé sur Android
  Cas 4 — updateProgressNotification → debounce (pas de spam si < 10% de différence)
  Cas 5 — dismissProgressNotification → dismissNotificationAsync appelé
  Cas 6 — requestPermissions granted → retourne true
  Cas 7 — requestPermissions denied → retourne false, téléchargement continue
  ```
- [ ] Subtask 8.3 : Mock de `expo-notifications` via jest manual mocks

### Task 9 : Validation finale (AC9)

- [ ] Subtask 9.1 : `npm run test:unit` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 9.2 : `npm run test:acceptance` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 9.3 : Test manuel sur device Android : lancer un téléchargement LLM, quitter l'app, vérifier progression en background + notification finale
- [ ] Subtask 9.4 : Test manuel sur device iOS : même scénario (vérifier URLSession background)
- [ ] Subtask 9.5 : Fermer l'issue GitHub #18 avec référence au commit

## Scénarios BDD (Feature File)

```gherkin
# language: fr
Fonctionnalité: Téléchargement de Modèles en Arrière-Plan
  En tant qu'utilisateur souhaitant télécharger un modèle LLM ou Whisper
  Je veux que le téléchargement continue même quand l'application passe en arrière-plan
  Afin de pouvoir utiliser d'autres applications et être notifié quand c'est terminé

  Contexte:
    Étant donné que je suis un utilisateur authentifié
    Et que les permissions de notification sont accordées

  Scénario: Notification de succès après téléchargement LLM terminé
    Étant donné que le téléchargement du modèle "Qwen2.5 3B" est en cours
    Quand le téléchargement se termine avec succès
    Alors une notification locale est planifiée avec le titre "Modèle téléchargé"
    Et le corps de la notification contient "Qwen2.5 3B est prêt à l'emploi"
    Et la notification contient les données de navigation vers "llm"

  Scénario: Notification d'échec après téléchargement LLM en erreur
    Étant donné que le téléchargement du modèle "Qwen2.5 3B" est en cours
    Quand le téléchargement échoue avec une erreur réseau
    Alors une notification locale est planifiée avec le titre "Échec du téléchargement"
    Et le corps de la notification contient "Qwen2.5 3B — Appuyer pour réessayer"
    Et la notification contient les données de navigation et retry

  Scénario: Notification de succès après téléchargement Whisper terminé
    Étant donné que le téléchargement du modèle Whisper "small" est en cours
    Quand le téléchargement se termine avec succès
    Alors une notification locale est planifiée avec le titre "Modèle téléchargé"
    Et la notification contient les données de navigation vers "whisper"

  Scénario: Permissions non accordées — téléchargement continue quand même
    Étant donné que les permissions de notification sont refusées par l'utilisateur
    Quand je lance le téléchargement d'un modèle LLM
    Alors le téléchargement démarre normalement
    Et aucune notification n'est envoyée
    Et aucune erreur n'est propagée à l'UI

  Scénario: Rafraîchissement état UI au retour dans l'app
    Étant donné qu'un téléchargement LLM s'est terminé en arrière-plan
    Quand l'application revient au premier plan
    Et que je navigue vers LLMSettingsScreen
    Alors le modèle est affiché comme "Downloaded"
    Et le bouton "Select" est disponible

  Scénario: Mise à jour notification progression Android
    Étant donné que je suis sur Android
    Et que le téléchargement d'un modèle est en cours à 20%
    Quand la progression atteint 30%
    Alors la notification persistante est mise à jour avec la nouvelle progression
    Et l'ancienne notification est remplacée (pas de doublon)
```

## Dev Notes

### 🔑 Fichiers à modifier

```
pensieve/mobile/src/contexts/Normalization/services/LLMModelService.ts
  ← Task 2: Injecter IModelDownloadNotificationService
  ← Task 2: Ajouter appels notifications dans done/error/progress handlers

pensieve/mobile/src/contexts/Normalization/services/TranscriptionModelService.ts
  ← Task 3: Migrer de expo/fetch streaming vers react-native-background-downloader
  ← Task 3: Ajouter pause/resume/cancel + recovery Whisper

pensieve/mobile/src/screens/settings/LLMSettingsScreen.tsx
  ← Task 6: Ajouter AppState listener pour refresh au retour

pensieve/mobile/src/screens/settings/WhisperSettingsScreen.tsx
  ← Task 6: Ajouter AppState listener pour refresh au retour

pensieve/mobile/app.json
  ← Task 4: Ajouter UIBackgroundModes pour iOS si absent

pensieve/mobile/src/infrastructure/di/container.ts
  ← Task 1: Enregistrer ModelDownloadNotificationService
```

### 📁 Fichiers à créer

```
pensieve/mobile/src/contexts/Normalization/domain/IModelDownloadNotificationService.ts
  ← Task 1.2: Interface

pensieve/mobile/src/contexts/Normalization/services/ModelDownloadNotificationService.ts
  ← Task 1: Implémentation expo-notifications

pensieve/mobile/src/contexts/Normalization/services/__tests__/ModelDownloadNotificationService.test.ts
  ← Task 8: Tests unitaires

pensieve/mobile/tests/acceptance/features/story-8-7-download-modeles-en-background.feature
  ← Task 7: Scénarios Gherkin

pensieve/mobile/tests/acceptance/story-8-7-download-modeles-en-background.test.ts
  ← Task 7: Step definitions BDD
```

### ⛔ Fichiers à NE PAS modifier

```
pensieve/mobile/src/contexts/Normalization/services/llmModelsConfig.ts
  ← Configuration statique — pas de changement requis

pensieve/mobile/src/components/llm/LLMModelCard.tsx
  ← UI existante avec boutons Download/Pause/Resume — pas de changement (la progression
    vient déjà du service)

pensieve/mobile/src/components/whisper/WhisperModelCard.tsx
  ← Même raison
```

### 🏗️ Architecture — Flux de notification

```
LLMModelService / TranscriptionModelService
  └─ download task (.done / .error / .progress)
       │
       ▼
IModelDownloadNotificationService
  ├─ notifyDownloadSuccess()  → Notifications.schedulePushNotificationAsync()
  ├─ notifyDownloadError()    → Notifications.schedulePushNotificationAsync()
  ├─ updateProgressNotification() → Notifications.presentNotificationAsync() [Android]
  └─ dismissProgressNotification() → Notifications.dismissNotificationAsync()
       │
       ▼
App.tsx — NotificationResponseReceivedListener
  └─ navigation.navigate('Settings', { screen: 'LLMSettings' | 'WhisperSettings' })
```

### 🏗️ Architecture — Background Download

**Android** (`@kesha-antonov/react-native-background-downloader` v4.x) :
- La bibliothèque crée automatiquement un **DownloadManager foreground service** sur Android
- Les téléchargements persistent dans le background sans configuration supplémentaire
- La notification persistante peut être fournie via les options Android native notification de la librairie **OU** via `expo-notifications` (vérifier l'API v4.x)

**iOS** (`@kesha-antonov/react-native-background-downloader` v4.x) :
- Utilise **NSURLSession avec background configuration** nativement
- Les téléchargements se poursuivent même si l'app est suspendue ou killed par iOS
- Limitation iOS : iOS peut terminer les tâches si la pression mémoire est trop élevée → la stratégie de resume (via `recoverInterruptedDownloads()`) est essentielle
- `UIBackgroundModes: ["fetch"]` dans `app.json` est requis

### 🏗️ Migration `TranscriptionModelService` — Points d'attention

**Avant** (expo/fetch streaming) :
- Progress granulaire basé sur les chunks ReadableStream
- SHA256 vérification sur le fichier complet en mémoire
- Pas de pause/resume natif

**Après** (react-native-background-downloader) :
- Progress via `bytesDownloaded / bytesTotal`
- SHA256 vérification **après** la fin du download (sur le fichier sur disque)
- Pause/resume natif supporté

**Compatibilité API ITranscriptionModelService** :
- Vérifier si l'interface existante expose `pauseDownload()` / `resumeDownload()` / `cancelDownload()`
- Si non → ajouter ces méthodes à l'interface (non-breaking, les implémentations existantes peuvent no-op)

### 📋 Conformité ADR

| ADR | Impact | Conformité |
|-----|--------|-----------|
| **ADR-021** (DI Lifecycle — Transient First) | `ModelDownloadNotificationService` → injectable, résoudre lazily dans `LLMModelService` (déjà injecté) | ✅ |
| **ADR-022** (AsyncStorage — non-critique seulement) | Permission state flag `@pensieve/notifications_requested` = préférence UI → conforme | ✅ |
| **ADR-023** (Result Pattern) | Les méthodes du service de notification peuvent retourner `void` (fire-and-forget acceptable pour notifications) ou `Result<void>` pour cohérence — privilégier `Result<void>` | ⚠️ Vérifier cohérence |
| **ADR-024** (Clean Code) | SRP : `ModelDownloadNotificationService` = 1 responsabilité (notifications) | ✅ |
| **ADR-018** (OP-SQLite) | Aucun impact | ✅ |

### 🚨 Points de vigilance techniques

#### 1. API `@kesha-antonov/react-native-background-downloader` v4.x

La version v4.x a une API différente de v2.x/v3.x. Vérifier **avant l'implémentation** :
- Nom de la fonction : `createDownloadTask()` ou `downloadFile()` ?
- Options disponibles pour background mode iOS
- Support de `task.pause()` / `task.resume()` dans v4.x

```typescript
// À vérifier dans node_modules/@kesha-antonov/react-native-background-downloader/src/index.d.ts
```

#### 2. iOS Background Mode — Limitation temporelle

iOS limite le temps de background execution. Pour les gros modèles (ex: Whisper large-v3 = 3.1Go), le téléchargement peut prendre plus d'une heure. L'URLSession background d'iOS gère cela automatiquement, mais le **hook de recovery** (`useLLMDownloadRecovery`) doit être robuste pour reprendre si iOS tue l'app.

#### 3. Permissions notifications — Timing

Ne pas demander les permissions avant une interaction utilisateur. Le bon moment = **quand l'utilisateur lance son premier téléchargement** (pas au démarrage de l'app).

#### 4. Pas de WiFi-Only dans cette story

L'issue #18 mentionne "Possibilité de limiter aux WiFi uniquement" — c'est marqué comme optionnel. Cette story se concentre sur le background download + notifications. La feature WiFi-only peut être une story séparée si nécessaire.

### 🎯 Stratégie de test

Les tests BDD simuleront le comportement background en mockant les callbacks `done`/`error` du `react-native-background-downloader`. On ne peut pas vraiment tester le "app en background" dans un environnement Jest — le focus est sur :
1. Que les callbacks appellent les bonnes méthodes de notification
2. Que le service de notification planifie les bonnes notifications
3. Que l'AppState listener rafraîchit l'UI correctement

### 📋 Vérifications pré-implémentation

Avant de commencer le code, vérifier dans le projet :

1. **`ITranscriptionModelService` interface** — quelles méthodes expose-t-elle actuellement ?
2. **`app.json`** — `UIBackgroundModes` déjà configuré ?
3. **`App.tsx` ou navigation root** — quel est le pattern de navigation ref existant ?
4. **`TOKENS`** — vérifier l'objet TOKENS dans `container.ts` pour le pattern d'ajout de nouveau token

### Project Structure Notes

```
pensieve/mobile/src/
├── contexts/Normalization/
│   ├── domain/
│   │   ├── ILLMModelService.ts              ← Existant
│   │   ├── ITranscriptionModelService.ts   ← VÉRIFIER méthodes exposées
│   │   └── IModelDownloadNotificationService.ts ← CRÉER : interface
│   └── services/
│       ├── LLMModelService.ts              ← MODIFIER : add notifications
│       ├── TranscriptionModelService.ts    ← MODIFIER : migrer vers bg-downloader
│       ├── HuggingFaceAuthService.ts       ← NE PAS MODIFIER
│       ├── llmModelsConfig.ts              ← NE PAS MODIFIER
│       └── ModelDownloadNotificationService.ts ← CRÉER : implémentation
├── infrastructure/di/container.ts          ← MODIFIER : enregistrer nouveau service
├── screens/settings/
│   ├── LLMSettingsScreen.tsx              ← MODIFIER : AppState listener
│   └── WhisperSettingsScreen.tsx          ← MODIFIER : AppState listener
└── app.json                                ← MODIFIER si nécessaire : UIBackgroundModes

tests/acceptance/
├── features/
│   └── story-8-7-download-modeles-en-background.feature ← CRÉER
└── story-8-7-download-modeles-en-background.test.ts     ← CRÉER
```

### References

- [Source: GitHub Issue #18](https://github.com/yohikofox/pensieve/issues/18) — Spécification originale complète
- [Source: `mobile/src/contexts/Normalization/services/LLMModelService.ts`] — Service LLM existant avec `react-native-background-downloader`
- [Source: `mobile/src/contexts/Normalization/services/TranscriptionModelService.ts`] — Service Whisper à migrer (expo/fetch streaming)
- [Source: `mobile/src/contexts/Normalization/services/HuggingFaceAuthService.ts`] — Auth service pour headers
- [Source: `mobile/src/hooks/initialization/useLLMDownloadRecovery.ts`] — Pattern recovery existant à aligner
- [Source: `mobile/src/stores/settingsStore.ts`] — Pattern AsyncStorage settings
- [Source: `_bmad-output/implementation-artifacts/stories/epic-8/8-6-transcription-live-avec-waveform.md`] — Story précédente (patterns DI, hooks, notification service)
- [Source: ADR-021] — DI Lifecycle (Transient First)
- [Source: ADR-022] — AsyncStorage (données non-critiques uniquement)
- [Source: ADR-023] — Result Pattern
- [Source: expo-notifications docs] — API locale notifications (permissions, scheduling, channels)
- [Source: @kesha-antonov/react-native-background-downloader v4.x] — Vérifier API exacte dans node_modules avant implémentation

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #18 — analyse exhaustive : LLMModelService (react-native-background-downloader déjà utilisé), TranscriptionModelService (expo/fetch streaming à migrer), expo-notifications + expo-background-fetch + expo-task-manager déjà dans deps. Architecture décidée : ModelDownloadNotificationService nouveau, migration TranscriptionModelService, AppState listener pour sync UI. | yohikofox |
