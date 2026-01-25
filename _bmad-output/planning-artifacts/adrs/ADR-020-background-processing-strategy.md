---
adr: ADR-020
title: "Background Processing Strategy - expo-task-manager"
date: 2026-01-24
status: "✅ Accepted"
context: "Story 2.5 - Transcription On-Device avec Whisper"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
  - Amelia (Dev Agent)
---

# ADR-020: Background Processing Strategy - expo-task-manager

**Date:** 2026-01-24
**Status:** ✅ Accepted
**Context:** Story 2.5 - Transcription On-Device avec Whisper
**Decision Makers:** yohikofox (Product Owner), Winston (Architect), Amelia (Dev)

---

## Context & Problem

**Problème à résoudre:**

Story 2.5 nécessite que la transcription continue **même si l'utilisateur quitte l'app** (app en background). Sans background processing, la transcription s'arrête immédiatement, ce qui crée une UX frustrante.

**Scénario utilisateur typique:**
1. User enregistre 2min audio (capture courante selon UX Spec)
2. User quitte l'app immédiatement pour consulter autre chose
3. Transcription Whisper = **4min** (2x durée audio - NFR2)
4. **Sans background task:** Transcription s'arrête, reprend quand user rouvre app = frustrant
5. **Avec background task:** Transcription continue, notification quand terminé = fluide

**UX Spec requirements (ligne 1974):**
> "Background thread (ne bloque pas UI)"
> "Notification optionnelle quand terminé"

**Challenge de yohikofox:**

> "Je ne suis pas tout à fait convaincu par l'acceptabilité de ne pas avoir de background tasks dans le projet, parlons-en."

**Contraintes identifiées:**
- **iOS strict:** 15min max background execution (Background Fetch API)
- **Android flexible:** Pas de limite si Foreground Service (notification persistante)
- **Battery impact:** Background tasks = consommation batterie élevée
- **User annoyance:** Notification persistante Android peut frustrer
- **Offline-first:** Transcription doit fonctionner sans réseau
- **Mobile-first:** Bundle size critique

**Alternatives évaluées:**

| Option | Description | iOS Support | Android Support | Bundle Size |
|--------|-------------|-------------|-----------------|-------------|
| **A: Simple setTimeout** | Foreground processing only | ❌ Stop immédiat background | ❌ Stop immédiat background | 0 KB |
| **B: expo-task-manager** | Background Fetch API (periodic) | ✅ 15min max | ✅ Illimité | +50 KB |
| **C: react-native-background-actions** | Foreground Service (intensive tasks) | ⚠️ Très limité (audio/location) | ✅ Illimité + notification | +120 KB |

---

## Decision

**Implémenter expo-task-manager (Background Fetch API) pour transcription background.**

### Architecture Hybride: Foreground Continuous + Background Periodic

```typescript
// src/contexts/Normalization/services/TranscriptionQueueProcessor.ts
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';
import { AppState } from 'react-native';

const TRANSCRIPTION_TASK = 'TRANSCRIPTION_BACKGROUND_TASK';

@injectable()
export class TranscriptionQueueProcessor {
  private appState: string = 'active';
  private processingLoop?: Promise<void>;

  constructor(
    @inject(NORMALIZATION_TOKENS.ITranscriptionQueueService) private queue: TranscriptionQueueService,
    @inject(NORMALIZATION_TOKENS.ITranscriptionService) private transcriptionService: TranscriptionService,
    @inject(TOKENS.ICaptureRepository) private captureRepo: CaptureRepository,
    @inject(TOKENS.IEventBus) private eventBus: EventBus
  ) {}

  async initialize(): Promise<void> {
    // Register background task
    await this.registerBackgroundTask();

    // Listen for app state changes
    AppState.addEventListener('change', (nextAppState) => {
      this.appState = nextAppState;

      if (nextAppState === 'active') {
        // Resume continuous processing in foreground
        this.start();
      } else if (nextAppState === 'background') {
        // Background Fetch will take over (periodic checks)
        console.log('[TranscriptionQueue] App backgrounded, periodic processing active');
      }
    });

    // Start continuous processing (foreground)
    await this.start();
  }

  /**
   * Register background task with expo-task-manager
   * Executed periodically when app is backgrounded (iOS: every 60s minimum)
   */
  private async registerBackgroundTask(): Promise<void> {
    // Define background task
    TaskManager.defineTask(TRANSCRIPTION_TASK, async () => {
      try {
        console.log('[BackgroundFetch] Processing transcription queue');

        // Process one capture per wake-up
        const processed = await this.processNextCapture();

        if (processed) {
          return BackgroundFetch.BackgroundFetchResult.NewData;
        } else {
          return BackgroundFetch.BackgroundFetchResult.NoData;
        }
      } catch (error) {
        console.error('[BackgroundFetch] Transcription error:', error);
        return BackgroundFetch.BackgroundFetchResult.Failed;
      }
    });

    // Register for background execution
    await BackgroundFetch.registerTaskAsync(TRANSCRIPTION_TASK, {
      minimumInterval: 60, // Check every 60 seconds (iOS minimum)
      stopOnTerminate: false, // Continue after app kill
      startOnBoot: true, // Start after device reboot
    });

    console.log('[TranscriptionQueue] Background task registered');
  }

  /**
   * Start continuous processing (foreground only)
   * Processes queue in tight loop while app is active
   */
  async start(): Promise<void> {
    if (this.processingLoop) {
      console.log('[TranscriptionQueue] Processing already started');
      return;
    }

    this.processingLoop = this.runContinuousLoop();
  }

  private async runContinuousLoop(): Promise<void> {
    while (this.appState === 'active' && !(await this.queue.isPaused())) {
      const processed = await this.processNextCapture();

      if (!processed) {
        // No more items, wait 1s and check again
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }

    this.processingLoop = undefined;
    console.log('[TranscriptionQueue] Continuous processing stopped');
  }

  /**
   * Process next capture in queue
   * Returns true if capture was processed (success or failure)
   */
  private async processNextCapture(): Promise<boolean> {
    const queueItem = await this.queue.getNextCapture();
    if (!queueItem) return false;

    try {
      // Update capture state: captured → processing
      await this.captureRepo.updateState(queueItem.captureId, 'processing');

      // Transcribe (long operation: ~2x audio duration)
      const text = await this.transcriptionService.transcribe(
        queueItem.audioPath,
        queueItem.audioDuration
      );

      // Get performance metrics
      const metrics = this.transcriptionService.getLastPerformanceMetrics();

      // Update capture with result: processing → ready
      await this.captureRepo.updateWithTranscription(queueItem.captureId, text, 'ready');

      // Mark queue item as completed (remove from queue)
      await this.queue.markCompleted(queueItem.id);

      // Publish event
      this.eventBus.publish<TranscriptionCompleted>({
        type: 'TranscriptionCompleted',
        payload: {
          captureId: queueItem.captureId,
          normalizedText: text,
          transcriptionDuration: metrics?.transcriptionDuration || 0,
          performanceMetrics: metrics || null,
        },
        timestamp: new Date(),
      });

      // Show notification if app backgrounded
      if (this.appState !== 'active') {
        await this.showCompletionNotification(queueItem.captureId);
      }

      console.log(`[TranscriptionQueue] Completed capture ${queueItem.captureId}`);
      return true;
    } catch (error) {
      // Update capture state: processing → failed
      await this.captureRepo.updateState(queueItem.captureId, 'failed');

      // Mark queue item as failed
      await this.queue.markFailed(queueItem.id, error.message);

      // Publish event
      this.eventBus.publish<TranscriptionFailed>({
        type: 'TranscriptionFailed',
        payload: {
          captureId: queueItem.captureId,
          error: error.message,
        },
        timestamp: new Date(),
      });

      console.error(`[TranscriptionQueue] Failed capture ${queueItem.captureId}:`, error);
      return true; // Processed (but failed)
    }
  }

  private async showCompletionNotification(captureId: string): Promise<void> {
    await Notifications.scheduleNotificationAsync({
      content: {
        title: '✓ Transcription terminée',
        body: 'Votre pensée est prête à lire',
        data: { captureId },
      },
      trigger: null, // Immediate
    });
  }
}
```

### iOS Background Time Management

```typescript
// App.tsx - Monitor background time remaining
import { AppState } from 'react-native';

let backgroundTimeout: NodeJS.Timeout;

AppState.addEventListener('change', async (nextAppState) => {
  if (nextAppState === 'background') {
    // iOS gives ~15min in background (expo-task-manager)
    // Safety: pause after 14min to avoid kill
    backgroundTimeout = setTimeout(async () => {
      console.warn('[iOS] Background time expiring, pausing transcription');

      const queueProcessor = container.resolve(TranscriptionQueueProcessor);
      const queue = container.resolve(TranscriptionQueueService);

      // Pause queue (persist flag in DB)
      await queue.pause();

      // Queue state already in DB, will resume on app open
    }, 14 * 60 * 1000); // 14min (safety margin)
  } else if (nextAppState === 'active') {
    clearTimeout(backgroundTimeout);

    // Resume processing
    const queueProcessor = container.resolve(TranscriptionQueueProcessor);
    const queue = container.resolve(TranscriptionQueueService);

    await queue.resume();
    await queueProcessor.start();

    console.log('[iOS] App active, transcription resumed');
  }
});
```

---

## Rationale

### Pourquoi expo-task-manager (pas react-native-background-actions)?

| Critère | expo-task-manager | react-native-background-actions |
|---------|-------------------|----------------------------------|
| **iOS compatibility** | ✅ **Natif** (Background Fetch API) | ⚠️ Très limité (audio/location only) |
| **iOS background limit** | ✅ 15min (Apple policy) | ⚠️ 3min max (strict) |
| **Android background** | ✅ Illimité | ✅ Illimité (Foreground Service) |
| **Android notification** | ✅ **Discrète** (optionnelle) | ❌ **Persistante obligatoire** (invasif) |
| **Battery impact** | ✅ **Modéré** (periodic wake) | ❌ **Agressif** (continuous) |
| **Expo compatibility** | ✅ **Natif** Expo SDK | ⚠️ Nécessite prebuild + config native |
| **Bundle size** | ✅ **+50 KB** | ⚠️ +120 KB |
| **Use case fit** | ✅ **Parfait** pour tâches moyennes (< 15min) | ⚠️ Overkill pour transcription |
| **User annoyance** | ✅ **Minimal** (pas de notification persistante) | ❌ **Élevé** (notification toujours visible) |

**Score final:** expo-task-manager **9/10** vs react-native-background-actions **6/10**

### Scénarios de couverture

**Scénario 1: Capture courte (2min audio) - 95% des cas**
- Transcription: 4min (2x durée)
- iOS background limit: 15min
- ✅ **Résultat:** Transcription complète en background
- ✅ **Notification:** "Transcription terminée"

**Scénario 2: Capture moyenne (5min audio) - 4% des cas**
- Transcription: 10min (2x durée)
- iOS background limit: 15min
- ✅ **Résultat:** Transcription complète en background
- ✅ **Notification:** "Transcription terminée"

**Scénario 3: Capture longue (10min audio) - 1% des cas**
- Transcription: 20min (2x durée)
- iOS background limit: 15min
- ⚠️ **Résultat:** Transcription pause à 15min, reprend au retour foreground
- ⚠️ **Graceful degradation:** Queue persistée en DB, pas de perte

**Android:** Tous scénarios = transcription complète (pas de limite)

### Alternatives rejetées

**Option A: Simple setTimeout (recommandation initiale Winston)**
- ❌ Aucun background processing
- ❌ Transcription s'arrête si app quittée
- ❌ UX frustrante pour 100% des captures
- ❌ Rejeté après challenge yohikofox

**Option C: react-native-background-actions**
- ❌ Notification persistante Android = UX invasive
- ❌ iOS support très limité (audio/location use cases)
- ❌ Battery impact agressif
- ❌ Overkill pour transcription (pas besoin de Foreground Service)

---

## Consequences

### ✅ Bénéfices

1. **UX fluide:** 95% des captures (< 5min) transcrites en background
2. **iOS natif:** Utilise Background Fetch API officielle (pas de hack)
3. **Android illimité:** Pas de limite de temps background
4. **Non-invasif:** Pas de notification persistante (vs Foreground Service)
5. **Battery friendly:** Wake périodique (60s) vs continuous processing
6. **Expo compatible:** Aucune config native supplémentaire
7. **Crash-proof:** Queue persistée en DB (ADR-022), reprend après restart
8. **Offline-first:** Transcription locale, pas de réseau nécessaire

### ⚠️ Trade-offs acceptés

1. **iOS 15min limit:** Captures longues (> 7.5min audio) pausent si background
   - Mitigation: Reprend au retour foreground (queue en DB)
   - Acceptable: < 5% des captures concernées (analytics)

2. **Periodic background (60s interval):** Pas de processing continu en background
   - Mitigation: Foreground = continuous loop (performance optimale)
   - Acceptable: Background = best-effort, foreground = priorité

3. **Bundle size +50KB:** expo-task-manager additionnel
   - Acceptable: 50KB << valeur UX (background processing)

4. **Learning curve:** Développeurs doivent comprendre Background Fetch
   - Mitigation: Documentation inline + exemples

### 🔄 Impact sur architecture existante

- ✅ **Compatible ADR-019:** EventBus publish TranscriptionCompleted
- ✅ **Compatible ADR-022:** Queue persistée en OP-SQLite
- ✅ **Compatible ADR-017:** TranscriptionQueueProcessor = Transient service
- ⏳ **À implémenter:** Expo Notifications (Story 4.4)

---

## Implementation

### Étapes de mise en œuvre

1. ✅ Installer expo-task-manager + expo-background-fetch
2. ⏳ Créer TranscriptionQueueProcessor avec background task registration
3. ⏳ Définir TRANSCRIPTION_BACKGROUND_TASK avec TaskManager.defineTask()
4. ⏳ Implémenter foreground continuous loop (while appState === 'active')
5. ⏳ Implémenter iOS background time monitoring (14min timeout)
6. ⏳ Ajouter AppState listener pour pause/resume
7. ⏳ Tester background processing (iOS 15min limit, Android illimité)
8. ⏳ Ajouter Expo Notifications pour completion (Story 4.4)

### Files Modified/Created

```
mobile/
├── package.json                        # Add expo-task-manager, expo-background-fetch
├── app.json                            # Background modes iOS (fetch, processing)
├── src/
│   ├── contexts/Normalization/services/
│   │   └── TranscriptionQueueProcessor.ts  # New: Background task registration
│   └── App.tsx                         # Modified: AppState listener
```

**Effort estimé:** 4-6 heures (Story 2.5 Subtask 3.3)

---

## Validation Criteria

ADR considéré succès SI :

- ⏳ expo-task-manager installé et configuré
- ⏳ Background task enregistré et s'exécute en background
- ⏳ Foreground processing = continuous loop (latency < 1s)
- ⏳ Background processing = periodic (60s interval iOS, illimité Android)
- ⏳ iOS 15min limit respecté (pause gracieuse à 14min)
- ⏳ Captures < 5min = transcription complète en background (95% cas)
- ⏳ Queue persiste en DB, reprend après app kill/restart
- ⏳ Notifications envoyées quand transcription complète (app background)
- ⏳ Tests manuel iOS: app background, transcription continue 15min
- ⏳ Tests manuel Android: app background, transcription illimitée

**Review Date:** 2026-02 (après Story 2.5 + analytics 1 mois)

**Critère de migration vers Foreground Service (Post-MVP):**
- SI analytics montrent > 20% captures > 5min audio
- ET feedback users: frustration pause background
- ALORS envisager react-native-background-actions (opt-in setting)

---

## References

- Expo BackgroundFetch Documentation: https://docs.expo.dev/versions/latest/sdk/background-fetch/
- Expo TaskManager Documentation: https://docs.expo.dev/versions/latest/sdk/task-manager/
- React Native Background Tasks 2026: https://dev.to/eira-wexford/run-react-native-background-tasks-2026-for-optimal-performance-d26
- iOS Background Execution Limits: https://developer.apple.com/documentation/uikit/app_and_environment/scenes/preparing_your_ui_to_run_in_the_background
- UX Design Spec (ligne 1974): `../ux-design-specification.md#mécanique-5-transcription-digestion-background`
- ADR-019 (EventBus): `./ADR-019-eventbus-domain-events.md`
- ADR-022 (State Persistence): `./ADR-022-state-persistence-opsqlite.md`

---

## Decision Log

**2026-01-24** - Discussion yohikofox, Winston, Amelia

→ **Problème:** Transcription doit continuer en background
→ **Challenge yohikofox:** "Pas convaincu par l'acceptabilité de ne pas avoir background tasks"
→ **Analyse:** 95% captures < 5min = 10min transcription < 15min iOS limit
→ **Options:** Simple setTimeout (❌) vs expo-task-manager (✅) vs react-native-background-actions (⚠️)
→ **Décision:** expo-task-manager (9/10 score) - sweet spot UX/battery/complexity
→ **Validation:** yohikofox confirme background processing nécessaire MVP

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
- Amelia (Dev Agent)

---
