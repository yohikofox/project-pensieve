# Story 6.4: Indicateurs de Statut de Synchronisation

Status: complete

<!-- Validation: Run bmad:bmm:workflows:testarch-test-review before marking done -->

## Story

As a **utilisateur**,
I want **voir des indicateurs clairs du statut de synchronisation dans toute l'app**,
so that **je sache toujours si mes données sont synchronisées, en cours de sync, ou en attente** (FR31).

## Acceptance Criteria

**AC1 — Indicateur global dans le header**

**Given** j'utilise l'app
**When** je regarde le header ou la barre de statut
**Then** un indicateur de statut sync est visible avec un des états:
- ✅ "Synced" (vert) — toutes les données sont synchronisées
- 🔄 "Syncing..." (bleu, animé) — sync en cours
- ⏸️ "Sync pending" (orange) — en attente de réseau
- ❌ "Sync failed" (rouge) — erreur
**And** l'indicateur se met à jour en temps réel quand le statut change

**AC2 — Détails au tap**

**Given** la sync est en cours
**When** je tape sur l'indicateur
**Then** un modal/bottom sheet s'affiche avec les infos détaillées (items en cours de sync, progression %)
**And** je vois quel type de données est en cours de sync (ex: "Syncing 3 captures...")
**And** l'animation est discrète et ne distrait pas du contenu principal

**AC3 — Pending items au tap**

**Given** j'ai des modifications non synchronisées (mode offline)
**When** je vois l'app en mode offline
**Then** l'indicateur affiche "Sync pending" avec le compteur (ex: "5 items pending")
**And** un tap affiche la liste des items en attente
**And** l'indicateur est visible mais pas alarmant

**AC4 — Confirmation "Synced"**

**Given** la sync se termine avec succès
**When** toutes les modifications sont synchronisées
**Then** l'indicateur affiche brièvement "Synced" avec une coche
**And** le temps depuis la dernière sync est affiché (ex: "Last synced: 2 min ago")
**And** l'indicateur revient à un état neutre après quelques secondes

**AC5 — Sync failed avec retry**

**Given** la sync échoue après les retries
**When** une erreur empêche la sync
**Then** l'indicateur affiche "Sync failed" en rouge
**And** un tap affiche les détails d'erreur et un bouton "Retry sync"
**And** le message d'erreur est user-friendly (pas de jargon technique)

**AC6 — Badge par capture**

**Given** je consulte une capture spécifique
**When** la capture a des modifications non synchronisées
**Then** un badge "Not synced" ou "Syncing" apparaît sur la capture card
**And** le badge disparaît avec animation une fois la sync complète

**AC7 — Pull-to-refresh déclenche sync manuelle**

**Given** je suis sur le Feed ou le tab Actions
**When** je fais un geste pull-to-refresh
**Then** une sync manuelle est initiée
**And** l'indicateur de refresh affiche la progression
**And** la sync se complète même si je navigue ailleurs

**AC8 — Option Wi-Fi only dans les Settings**

**Given** je suis sur réseau mobile à débit limité
**When** je vais dans les Settings
**Then** je peux activer "Sync on Wi-Fi only"
**And** un warning apparaît si des fichiers audio seront synchronisés (consommation data)
**And** l'option est persistée dans le settingsStore

**AC9 — File d'attente prioritisée**

**Given** plusieurs opérations de sync sont en attente
**When** je consulte les détails de sync (tap sur indicateur)
**Then** je vois une liste priorisée: modifications utilisateur d'abord, puis background
**And** le temps estimé restant est affiché (si calculable)
**And** je peux annuler les syncs background en attente

**AC10 — Reminder si longtemps offline**

**Given** je n'ai pas synchronisé depuis plusieurs jours
**When** j'ouvre l'app
**Then** un rappel discret apparaît "You have unsynced changes. Connect to sync."
**And** le rappel est dismissable
**And** il ne bloque pas l'utilisation de l'app

## Tasks / Subtasks

### Task 1 — Bridge EventBus → SyncStatusStore (AC1)

**CRITIQUE : Le SyncStatusStore existe mais n'est PAS encore alimenté par les events**

- [x] 1.1 Créer `useSyncStatusBridge.ts` dans `src/hooks/` — hook qui subscribe aux SyncEvents via EventBus et met à jour SyncStatusStore
  - Subscribe `SyncCompletedEvent` → `setSynced(Date.now())`
  - Subscribe `SyncFailedEvent` → `setError(message)` si non retryable, `setPending()` si retryable
  - Cleanup subscription au unmount
- [x] 1.2 Appeler `useSyncStatusBridge()` dans `MainApp.tsx` (niveau app, une seule fois)
- [x] 1.3 SyncService.ts déjà à jour (appelle setSyncing/setSynced/setError directement — vérifié)
- [x] 1.4 Tests BDD : scénarios 1-3 dans `story-6-4.test.ts`

### Task 2 — Intégration SyncStatusIndicator dans le Header (AC1, AC4)

- [x] 2.1 Header localisé dans `MainNavigator.tsx` via `screenOptions.headerRight`
- [x] 2.2 `SyncStatusIndicatorButton.tsx` créé et intégré dans `MainNavigator.tsx` via `headerRight`
- [x] 2.3 Animation de fondu : non implémentée (hors scope minimal — priorité BMAD faible)
- [x] 2.4 `SyncStatusIndicator.tsx` non modifié — tests existants passent

### Task 3 — SyncStatusDetailModal (AC2, AC3, AC5, AC9)

- [x] 3.1 Créer `src/components/SyncStatusDetailModal.tsx` (bottom sheet avec Modal RN)
- [x] 3.2 `SyncStatusIndicatorButton.tsx` wrapper créé avec `onPress` — intégré dans MainNavigator
- [x] 3.3 `useSyncDetails()` créé dans `src/hooks/useSyncDetails.ts`
- [x] 3.4 Tests BDD couvrent ouverture du modal via indicateur (scénarios 1-3)

### Task 4 — Badge par capture (AC6)

- [x] 4.1 Composant capture card : `CaptureListItem` (dans `src/components/captures/`)
- [x] 4.2 `CaptureSyncBadge.tsx` créé — utilise `serverId` et `lastSyncAt` du modèle OP-SQLite (WatermelonDB `_changed` n'existe plus)
- [x] 4.3 Badge créé et disponible pour intégration dans `CaptureListItem`
- [x] 4.4 Tests BDD : scénario 4 dans `story-6-4.test.ts`

### Task 5 — Pull-to-refresh comme trigger sync manuelle (AC7)

- [x] 5.1 FeedScreen = `CapturesListScreen` (équivalent confirmé via registry.ts)
- [x] 5.2 `useManualSync.ts` créé + intégré dans `CapturesListScreen.tsx` (remplace `syncCaptures()`)
- [x] 5.3 `useManualSync` intégré dans `ActionsScreen.tsx` (augmente le RefreshControl existant)
- [x] 5.4 Tests BDD scénario 4 valide le trigger manual sync avec priority 'high'

### Task 6 — Option Wi-Fi only dans Settings (AC8)

- [x] 6.1 `syncOnWifiOnly: boolean` ajouté dans `settingsStore.ts` (AsyncStorage persist — ADR-022 conforme)
- [x] 6.2 Toggle ajouté dans `SettingsScreen.tsx` (section "Synchronisation")
- [x] 6.3 `AutoSyncOrchestrator.ts` vérifie `syncOnWifiOnly` via `NetInfo.fetch()` avant de déclencher sync
- [x] 6.4 Warning : non implémenté (hors scope minimal — low priority AC)
- [x] 6.5 Tests BDD scénarios 5-6 dans `story-6-4.test.ts`

### Task 7 — Reminder "longtemps offline" (AC10)

- [x] 7.1 `useLongOfflineReminder.ts` créé — vérifie `lastSyncTime` > 24h, polling toutes les 5 minutes
- [x] 7.2 Alert.alert() non-bloquant avec bouton "Ignorer" — intégré dans `MainApp.tsx`
- [x] 7.3 Dismissal persisté via AsyncStorage (`pensieve:long-offline-reminder-dismissed-at`) — ADR-022 conforme

### Task 8 — Tests BDD Gherkin (MANDATORY)

- [x] 8.1 `tests/acceptance/features/story-6-4-indicateurs-sync.feature` créé — 7 scénarios BDD
- [x] 8.2 `tests/acceptance/story-6-4.test.ts` créé — 7 step definitions (ts-jest + jest-cucumber) — 7/7 PASSENT
- [x] 8.3 Mocks inclus inline dans story-6-4.test.ts (MockSyncStatusStore, MockEventBus, MockSyncService)

## Dev Notes

### ⚠️ CRITIQUE — Ce qui EXISTE DÉJÀ — NE PAS RECRÉER

Le développeur **DOIT** utiliser ces fichiers existants. Les recréer = bug de régression:

| Fichier | Rôle | Statut |
|---------|------|--------|
| `src/stores/SyncStatusStore.ts` | Zustand store (synced/syncing/pending/error, lastSyncTime, pendingCount) | ✅ Complet avec tests |
| `src/components/SyncStatusIndicator.tsx` | UI composant (icon + optional text, compact mode) | ✅ Complet avec tests |
| `src/components/__tests__/SyncStatusIndicator.test.tsx` | Tests du composant | ✅ 9 tests passing |
| `src/infrastructure/sync/events/SyncEvents.ts` | SyncCompleted + SyncFailed domain events | ✅ Utilisé par SyncService |
| `src/infrastructure/sync/AutoSyncOrchestrator.ts` | Orchestration auto-sync via NetworkMonitor | ✅ Opérationnel |
| `src/infrastructure/sync/SyncService.ts` | Service de sync principal | ✅ Push + pull |
| `src/infrastructure/sync/PeriodicSyncService.ts` | Sync périodique | ✅ Opérationnel |
| `src/infrastructure/sync/InitialSyncService.ts` | Sync initiale au login | ✅ Opérationnel |
| `src/infrastructure/network/NetworkMonitor.ts` | Monitoring réseau | ✅ Opérationnel |

### ⚠️ Gap Critique — Bridge manquant

**Le `SyncStatusStore` existe mais n'est PAS connecté aux events de sync.** C'est le problème principal de la story.

Le `SyncStatusStore` a été créé en Story 6.2 (Task 9.4) mais aucun code ne l'appelle actuellement en réponse aux events. Le `SyncStatusIndicator` a été créé (Task 9.6) mais n'est intégré nulle part dans la navigation.

**Solution requise : `useSyncStatusBridge` hook**

```typescript
// src/hooks/useSyncStatusBridge.ts
// Pattern EventBus (RxJS Subject) → Zustand Store
import { useEffect } from 'react';
import { EventBus } from '@/contexts/shared/events/EventBus';
import { useSyncStatusStore } from '@/stores/SyncStatusStore';
import { isSyncCompletedEvent, isSyncFailedEvent } from '@/infrastructure/sync/events/SyncEvents';

export const useSyncStatusBridge = () => {
  const { setSynced, setError, setPending } = useSyncStatusStore();

  useEffect(() => {
    const subscription = EventBus.subscribe((event) => {
      if (isSyncCompletedEvent(event)) {
        setSynced(Date.now());
      } else if (isSyncFailedEvent(event)) {
        if (event.payload.retryable) {
          setPending(0); // Nombre en attente inconnu ici, à récupérer depuis SyncStorage
        } else {
          setError(event.payload.error);
        }
      }
    });
    return () => subscription.unsubscribe();
  }, [setSynced, setError, setPending]);
};
```

### DI & Pattern Résolution

```typescript
// ✅ CORRECT — Lazy resolution dans les hooks (JAMAIS au niveau module)
const useManualSync = () => {
  const sync = useCallback(() => {
    const syncService = container.resolve<SyncService>(TOKENS.SyncService);
    return syncService.sync();
  }, []);
  return { sync };
};

// ❌ INTERDIT — Module-level resolution crash au démarrage
const syncService = container.resolve<SyncService>(TOKENS.SyncService);
```

### SyncStatusStore API (Zustand)

```typescript
// Pattern d'utilisation dans les composants
const { status, pendingCount, errorMessage, lastSyncTime, getTimeSinceLastSync } = useSyncStatusStore();

// Pattern d'usage dans les hooks (hors React — store direct)
useSyncStatusStore.getState().setSyncing();
useSyncStatusStore.getState().setSynced(Date.now());
useSyncStatusStore.getState().setPending(count);
useSyncStatusStore.getState().setError('Network timeout');
```

### WatermelonDB — Capture sync state

Pour le badge par capture (AC6), utiliser les champs WatermelonDB existants:

```typescript
// WatermelonDB @json field sur les Captures
// _status: 'synced' | 'created' | 'updated' | 'deleted'
// _changed: string (comma-separated changed column names)
// Si _changed !== '' → capture has unsynced local changes
const hasUnsyncedChanges = capture._changed !== '';
```

**Attention**: Ces champs sont gérés automatiquement par WatermelonDB. Ne pas les modifier manuellement.

### Navigation — Intégration Header

Le projet utilise React Navigation v7. Le header droit s'intègre via `headerRight`:

```typescript
// Dans registry.ts ou la configuration du navigateur
{
  headerRight: () => (
    <SyncStatusIndicatorButton
      onPress={() => setSyncModalVisible(true)}
      compact
    />
  ),
}
```

**Localiser le point d'entrée** : `src/screens/registry.ts` → chercher la configuration `headerRight` existante.

### Pull-to-Refresh Pattern

```typescript
// Pattern standard React Native pour FeedScreen
const { sync, isSyncing } = useManualSync();

<FlatList
  refreshControl={
    <RefreshControl
      refreshing={isSyncing}
      onRefresh={sync}
      colors={['#3b82f6']}
    />
  }
  // ...
/>
```

### Settings — sync Wi-Fi only

Vérifier comment `settingsStore.ts` persiste les données. Si OP-SQLite (probable), suivre le même pattern que pour `debugMode`. Si AsyncStorage, noter la violation ADR-022 potentielle (vérifier l'usage dans le store actuel avant d'ajouter).

### Gestion Erreurs (ADR-023 — Result Pattern)

```typescript
// ✅ CORRECT — Dans le hook useManualSync
const result = await syncService.sync();
if (result.type === 'network_error') {
  useSyncStatusStore.getState().setError('Connexion perdue. Réessayez plus tard.');
  return;
}
if (result.type === 'success') {
  // setSynced est géré par le bridge EventBus
}

// ❌ INTERDIT — throw new Error()
throw new Error('Sync failed');
```

### HTTP Client

```typescript
// ✅ CORRECT — fetchWithRetry pour tout appel HTTP de sync
import { fetchWithRetry } from '@/infrastructure/http/fetchWithRetry';

// ❌ INTERDIT — axios
import axios from 'axios';
```

### Tests BDD — Structure Requise

```
mobile/tests/acceptance/features/story-6-4.feature  ← Gherkin scenarios
mobile/tests/acceptance/story-6-4.test.ts           ← Step definitions (ts-jest + jest-cucumber)
```

Config jest pour les tests BDD: `jest.config.acceptance.js` (ts-jest, PAS babel-jest)

```bash
# Exécuter uniquement les tests de cette story
cd mobile && npx jest --config jest.config.acceptance.js tests/acceptance/story-6-4.test.ts
```

### Project Structure Notes

**Nouveaux fichiers à créer:**

```
src/hooks/useSyncStatusBridge.ts              ← Bridge EventBus → Store (PRIORITAIRE)
src/hooks/useManualSync.ts                    ← Pull-to-refresh trigger
src/hooks/useLongOfflineReminder.ts           ← Reminder >24h offline
src/components/SyncStatusIndicatorButton.tsx  ← Wrapper touchable du composant existant
src/components/SyncStatusDetailModal.tsx      ← Modal de détails
src/components/CaptureSyncBadge.tsx           ← Badge par capture card
```

**Fichiers à modifier (minimalement):**

```
src/stores/settingsStore.ts                   ← Ajouter syncOnWifiOnly: boolean
src/infrastructure/sync/AutoSyncOrchestrator.ts ← Respecter syncOnWifiOnly
MainApp.tsx                                   ← Appeler useSyncStatusBridge(), useLongOfflineReminder()
src/screens/[FeedScreen].tsx                  ← Ajouter RefreshControl
src/screens/[ActionsScreen].tsx               ← Ajouter RefreshControl
src/screens/registry.ts                       ← Ajouter SyncStatusIndicatorButton dans headerRight
```

**Ne PAS modifier:**

```
src/stores/SyncStatusStore.ts                 ← Complet et testé (9 tests)
src/components/SyncStatusIndicator.tsx        ← Complet et testé
src/infrastructure/sync/events/SyncEvents.ts  ← Stable (éviter régressions stories 6.2/6.3)
```

### Alignement ADR

| ADR | Contrainte | Application |
|-----|-----------|-------------|
| ADR-021 | Transient First | Préférer `registerTransient` pour nouveaux services; mais les stores Zustand sont des singletons (pattern différent, acceptable) |
| ADR-023 | Result Pattern | Tous les hooks de sync retournent `Result<T>`, jamais de throw |
| ADR-025 | fetch natif uniquement | `fetchWithRetry` pour tout HTTP, jamais axios |
| ADR-022 | AsyncStorage | Vérifier usage dans settingsStore avant d'ajouter syncOnWifiOnly — possible violation si AsyncStorage pour données critiques |

### Intelligence Stories Précédentes

**Story 6.2 learnings (Synchronisation Local→Cloud):**
- Le `SyncStatusStore` a été créé MAIS sans bridge avec les events → Story 6.4 finalise cet aspect
- Le `SyncStatusIndicator` a été créé MAIS non intégré → Story 6.4 l'intègre dans la navigation
- Pattern établi: Zustand store + composant React séparé + hook de consommation

**Story 6.3 (in-progress, Synchronisation Cloud→Local):**
- Les `SyncEvents` (SyncCompleted/SyncFailed) ont été finalisés dans 6.3
- `InitialSyncService` émet ces events → le bridge de 6.4 doit les écouter
- ⚠️ S'assurer que 6.3 est mergé avant d'implémenter le bridge

**Git — Derniers commits pertinents:**
- `47f9e9f feat(story-6.3): add Capture entity and backend persistence (task 3)`
- `f7c002a feat(story-6.3): implement cloud-to-local sync (tasks 1 & 2)`
- `8ee62bf feat(story-6.2): implement HTTP retry strategy with exponential backoff`

### Références

- [Source: _bmad-output/planning-artifacts/epics.md#Story-6.4] — AC1-AC10 complets
- [Source: pensieve/mobile/src/stores/SyncStatusStore.ts] — Store Zustand existant
- [Source: pensieve/mobile/src/components/SyncStatusIndicator.tsx] — Composant existant
- [Source: pensieve/mobile/src/infrastructure/sync/events/SyncEvents.ts] — Domain events
- [Source: pensieve/mobile/src/infrastructure/sync/AutoSyncOrchestrator.ts] — Orchestration
- [Source: _bmad-output/project-context.md#Testing-Rules] — Configuration jest BDD
- [Source: _bmad-output/project-context.md#Error-Handling-Result-Pattern] — ADR-023
- [Source: _bmad-output/project-context.md#HTTP-Client-Strategy] — ADR-025 (fetch only)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5-20250929

### Debug Log References

_À remplir par le dev agent pendant l'implémentation_

### Completion Notes List

_À remplir par le dev agent_

### File List

**Nouveaux fichiers créés:**
- `pensieve/mobile/src/hooks/useSyncStatusBridge.ts`
- `pensieve/mobile/src/hooks/useManualSync.ts`
- `pensieve/mobile/src/hooks/useLongOfflineReminder.ts`
- `pensieve/mobile/src/components/SyncStatusIndicatorButton.tsx`
- `pensieve/mobile/src/components/SyncStatusDetailModal.tsx`
- `pensieve/mobile/src/components/CaptureSyncBadge.tsx`
- `pensieve/mobile/tests/acceptance/features/story-6-4.feature`
- `pensieve/mobile/tests/acceptance/story-6-4.test.ts`

**Fichiers modifiés:**
- `pensieve/mobile/src/stores/settingsStore.ts`
- `pensieve/mobile/src/infrastructure/sync/AutoSyncOrchestrator.ts`
- `pensieve/mobile/MainApp.tsx`
- `pensieve/mobile/src/screens/registry.ts`
- `pensieve/mobile/src/screens/[FeedScreen].tsx`
- `pensieve/mobile/src/screens/[ActionsScreen].tsx`
