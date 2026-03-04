# Story 16.3: Queue d'Analyses Asynchrone

Status: ready-for-dev

## Story

En tant qu'**utilisateur**,
je veux **que les demandes d'analyse LLM s'exécutent en file d'attente séquentielle en arrière-plan**,
afin de **ne plus subir de crashs lors d'analyses simultanées, de recevoir une notification à la complétion, et de continuer à naviguer pendant le traitement**.

## Contexte & Problème

**Bug identifié le 2026-03-04 :** L'app crashe lors de l'analyse d'une capture texte longue.

**Root cause :** `CaptureAnalysisService.analyze()` est appelé directement et sans mutex depuis `useAnalyses`. Quand plusieurs appels concurrents arrivent (ex: double-tap "Analyser tout", ou appel simultané depuis deux captures), `LitertLmBackend.unloadModel()` est déclenché pendant que `generate()` tourne côté natif Android → crash du module natif LiteRT-LM.

**Séquence de crash observée :**
```
[CaptureAnalysisService] Calling LLM...          ← inférence en cours
[LitertLmBackend] Model unloaded                 ← second appel concurrent → unload()
[LitertLmBackend] Loading model: ...             ← rechargement pendant inférence active
[LitertLmBackend] Error unloading model: Engine is not initialized
→ crash natif Android
```

**Ce que l'utilisateur veut (session 2026-03-04) :**
> "Il faudrait faire en sorte que cliquer sur le bouton demandant une analyse génère une demande d'analyse dans une queue, et que le service dédié dépile une par une les demandes d'analyse. Cette queue serait partagée entre toutes les captures et permettrait de tourner en arrière-plan et ferait office de notifications pour prévenir de la complétion."

## Acceptance Criteria

### AC1 — Enqueue au lieu d'exécution directe

**Given** l'utilisateur appuie sur "Analyser tout" ou un bouton d'analyse individuel
**When** la demande est soumise
**Then** une entrée est ajoutée à l'`AnalysisQueue` singleton
**And** l'action retourne immédiatement (non-bloquant)
**And** l'UI affiche un état "en queue" pour chaque type d'analyse demandé

### AC2 — Traitement séquentiel mono-thread

**Given** la queue contient N entrées d'analyse
**When** le worker de la queue traite les items
**Then** les analyses sont exécutées une par une (jamais en parallèle)
**And** `LitertLmBackend.generate()` n'est jamais appelé pendant un `unloadModel()` ou `loadModel()`
**And** aucun crash natif ne survient quelle que soit la cadence d'enqueue

### AC3 — Queue partagée inter-captures

**Given** l'utilisateur a demandé une analyse sur la capture A puis navigue vers la capture B et demande une analyse
**When** le worker traite la queue
**Then** les analyses de A sont traitées avant celles de B (FIFO)
**And** l'utilisateur peut voir dans l'UI que des analyses sont "en attente" même sur une autre capture

### AC4 — Notification de complétion par EventBus

**Given** une analyse se termine (succès ou échec)
**When** le worker a persisté le résultat
**Then** un événement `analysis.completed` (ou `analysis.failed`) est émis sur l'EventBus RxJS
**And** l'`AnalysisCard` dans l'écran détail de la bonne capture se met à jour automatiquement
**And** si l'utilisateur est sur une autre capture, la mise à jour se fait de façon silencieuse

### AC5 — Indicateur de progression dans l'UI

**Given** des analyses sont dans la queue ou en cours de traitement
**When** l'utilisateur ouvre l'écran détail d'une capture concernée
**Then** chaque type d'analyse en attente ou en cours affiche un spinner + label "En attente..." ou "En cours..."
**And** à la complétion, le contenu de l'analyse s'affiche sans rechargement manuel

### AC6 — Dédoublonnage des demandes

**Given** une analyse de type `summary` pour la capture X est déjà dans la queue ou en cours
**When** l'utilisateur redemande le même type d'analyse pour la même capture
**Then** la demande est ignorée (pas de doublon dans la queue)
**And** l'UI ne change pas d'état (déjà en queue)

### AC7 — Comportement en cas d'échec

**Given** une analyse échoue (timeout LLM, erreur modèle)
**When** le worker détecte l'échec
**Then** l'item est retiré de la queue (pas de retry automatique dans cette story)
**And** l'événement `analysis.failed` est émis sur l'EventBus
**And** l'UI affiche l'état d'erreur pour ce type d'analyse
**And** le worker passe à l'item suivant dans la queue

### AC8 — Tests unitaires et BDD

**Given** la queue et le worker sont implémentés
**When** les tests sont exécutés
**Then** tous les scénarios BDD passent (`npm run test:acceptance` dans `mobile/`)
**And** les tests unitaires couvrent : enqueue, déduplication, traitement séquentiel, émission d'événements

## Tasks / Subtasks

- [ ] **Task 1 — Créer `AnalysisQueueService` (AC1, AC2, AC3, AC6)**
  - [ ] 1.1 — Créer `mobile/src/contexts/Normalization/services/AnalysisQueueService.ts`
  - [ ] 1.2 — Implémenter l'interface : `enqueue(captureId, type)`, `getQueueStatus(captureId)`, `isInQueue(captureId, type)`
  - [ ] 1.3 — Worker interne : boucle séquentielle `while(queue.length) { await process(next) }` avec flag `isProcessing`
  - [ ] 1.4 — Dédoublonnage : vérifier `captureId+type` avant d'enqueuer
  - [ ] 1.5 — Enregistrer comme singleton dans `container.ts` via TSyringe

- [ ] **Task 2 — Intégrer EventBus pour les notifications de complétion (AC4)**
  - [ ] 2.1 — Définir les événements : `AnalysisCompletedEvent { captureId, analysisType, result }` et `AnalysisFailedEvent { captureId, analysisType, error }`
  - [ ] 2.2 — Émettre via `EventBus.publish('analysis.completed', ...)` à la fin de chaque item traité
  - [ ] 2.3 — Émettre `analysis.failed` sur erreur, puis continuer la queue (ne pas bloquer)

- [ ] **Task 3 — Adapter `useAnalyses` pour enqueuer plutôt qu'exécuter (AC1, AC5)**
  - [ ] 3.1 — Remplacer `analysisService.analyze()` par `queueService.enqueue(captureId, type)` dans `handleGenerateAnalysis`
  - [ ] 3.2 — Remplacer `analysisService.analyzeAll()` par 4 appels `queueService.enqueue()` dans `handleAnalyzeAll`
  - [ ] 3.3 — Adapter les états de chargement : `analysisLoading[type] = true` quand en queue, `false` à la complétion via EventBus
  - [ ] 3.4 — Souscrire à `analysis.completed` et `analysis.failed` dans `useAnalyses` via `useEffect` + `EventBus.on()`
  - [ ] 3.5 — Désabonnement propre dans le `return` du `useEffect` (éviter les fuites mémoire)

- [ ] **Task 4 — Indicateur "En attente" dans l'UI (AC5)**
  - [ ] 4.1 — Exposer `queueService.getQueueStatus(captureId)` depuis `useAnalyses` (ou via store)
  - [ ] 4.2 — Dans `AnalysisCard` : afficher spinner + "En attente..." si `isInQueue && !isLoading`
  - [ ] 4.3 — Différencier visuellement "en queue" (gris) vs "en cours" (bleu/spinner actif)

- [ ] **Task 5 — Tests BDD** (AC8)
  - [ ] 5.1 — Créer `mobile/tests/acceptance/features/story-16-3-analysis-queue.feature`
  - [ ] 5.2 — Créer `mobile/tests/acceptance/story-16-3.test.ts` avec step definitions
  - [ ] 5.3 — Scénarios : enqueue simple, dédoublonnage, traitement séquentiel, notification complétion, gestion erreur

- [ ] **Task 6 — Tests unitaires `AnalysisQueueService`** (AC8)
  - [ ] 6.1 — Créer `mobile/src/contexts/Normalization/services/__tests__/AnalysisQueueService.test.ts`
  - [ ] 6.2 — Tests : `enqueue` ajoute à la queue, `isInQueue` retourne true, doublon ignoré, worker séquentiel (mock CaptureAnalysisService), événements émis

## Dev Notes

### Architecture de l'`AnalysisQueueService`

```typescript
// Pattern cible — worker séquentiel avec flag de protection
@singleton()
@injectable()
export class AnalysisQueueService {
  private queue: Array<{ captureId: string; type: AnalysisType }> = [];
  private isProcessing = false;

  constructor(
    @inject(CaptureAnalysisService) private analysisService: CaptureAnalysisService,
    @inject(EventBusService) private eventBus: EventBusService,
  ) {}

  enqueue(captureId: string, type: AnalysisType): void {
    if (this.isInQueue(captureId, type)) return; // AC6 dédoublonnage
    this.queue.push({ captureId, type });
    this.processNext(); // démarre le worker si inactif
  }

  private async processNext(): Promise<void> {
    if (this.isProcessing || this.queue.length === 0) return;
    this.isProcessing = true;
    const item = this.queue.shift()!;
    try {
      const result = await this.analysisService.analyze(item.captureId, item.type);
      this.eventBus.publish('analysis.completed', { ...item, result });
    } catch (error) {
      this.eventBus.publish('analysis.failed', { ...item, error });
    } finally {
      this.isProcessing = false;
      this.processNext(); // traiter le suivant
    }
  }
}
```

### Fichiers à créer / modifier

| Fichier | Action |
|---------|--------|
| `mobile/src/contexts/Normalization/services/AnalysisQueueService.ts` | **Créer** — service queue + worker |
| `mobile/src/contexts/Normalization/services/__tests__/AnalysisQueueService.test.ts` | **Créer** — tests unitaires |
| `mobile/src/infrastructure/di/container.ts` | **Modifier** — enregistrer `AnalysisQueueService` en singleton |
| `mobile/src/infrastructure/di/tokens.ts` | **Modifier si nécessaire** — ajouter token si résolution par interface |
| `mobile/src/hooks/useAnalyses.ts` | **Modifier** — enqueue + subscribe EventBus |
| `mobile/src/components/capture/AnalysisCard.tsx` (ou équivalent) | **Modifier** — afficher état "en queue" |
| `mobile/tests/acceptance/features/story-16-3-analysis-queue.feature` | **Créer** — scénarios Gherkin |
| `mobile/tests/acceptance/story-16-3.test.ts` | **Créer** — step definitions |

### Patterns obligatoires à respecter

**TSyringe DI (ADR-021 — Transient First) :**
- `AnalysisQueueService` est un singleton (état partagé requis pour la queue)
- Enregistrement via `container.registerSingleton(AnalysisQueueService)` dans `container.ts`
- Résolution lazy dans les hooks : `container.resolve(AnalysisQueueService)` à l'intérieur du hook, jamais au niveau module

**EventBus existant (`src/contexts/shared/events/EventBus.ts`) :**
- Utiliser le pattern `EventBus.publish(eventName, payload)` déjà en place
- Souscrire dans `useEffect` avec cleanup : `const sub = EventBus.on('analysis.completed', handler); return () => sub.unsubscribe();`
- **Ne pas créer un nouveau système de notification** — l'EventBus RxJS existant suffit

**Result Pattern (ADR-023, `src/contexts/shared/domain/Result.ts`) :**
- `CaptureAnalysisService.analyze()` retourne déjà `AnalyzeResult` (union type `AnalysisResult | AnalysisError`)
- L'`AnalysisQueueService` doit propager ce résultat dans l'événement `analysis.completed`

**Pattern de test BDD (jest-cucumber) :**
- Features Gherkin dans `tests/acceptance/features/`
- Step definitions dans `tests/acceptance/story-16-3.test.ts`
- Config : `npm run test:acceptance` → `jest.config.acceptance.js`
- Référence récente : `tests/acceptance/story-8-23.test.ts` (pattern le plus récent)

### Attention — Pièges à éviter

1. **Ne pas appeler `analysisService.analyze()` directement** depuis l'UI — toujours passer par l'`AnalysisQueueService`
2. **`CaptureAnalysisService` reste inchangé** — le queue service est un wrapper, pas un refactoring du service
3. **Pas de retry automatique dans cette story** — AC7 dit explicitement : retirer l'item et émettre `analysis.failed`
4. **`isProcessing` est un flag interne au service**, pas dans Zustand — seul l'EventBus communique vers l'UI
5. **Le `useAnalyses` hook** : `analysisLoading[type]` doit passer à `false` dans le subscriber EventBus, pas dans le `finally` d'un `await`

### Contexte de crash (pour les tests)

Le crash se produisait parce que `handleAnalyzeAll` et `handleGenerateAnalysis` pouvaient être appelés en parallèle (double-tap, ou plusieurs boutons individuels). Chaque appel résolvait `container.resolve(CaptureAnalysisService)` (même singleton) et appelait `analyze()` simultanément. L'`AnalysisQueueService` résout ce problème structurellement : une seule inférence LLM à la fois.

### Référence — Commits récents pertinents

- `fix(story-16.2)` : `sync.service.ts` + `knowledge.module.ts` — pattern d'intégration inter-modules
- `refactor(ux): refonte écran détail capture — tabs Analyse/Transcription` — `CaptureDetailContent` restructuré avec tabs
- `fix(knowledge): séparer config ClientProxy et consumer RabbitMQ` — fix bug RabbitMQ du 2026-03-04

### Références

- [CaptureAnalysisService] `mobile/src/contexts/Normalization/services/CaptureAnalysisService.ts`
- [LitertLmBackend] `mobile/src/contexts/Normalization/services/postprocessing/LitertLmBackend.ts`
- [useAnalyses] `mobile/src/hooks/useAnalyses.ts`
- [EventBus] `mobile/src/contexts/shared/events/EventBus.ts`
- [Result Pattern] `mobile/src/contexts/shared/domain/Result.ts`
- [container.ts] `mobile/src/infrastructure/di/container.ts`
- [Story 16.1] `_bmad-output/implementation-artifacts/stories/epic-16/16-1-capture-processing-guards-feedback.md` — guards pendant processing
- [Story 16.2] `_bmad-output/implementation-artifacts/stories/epic-16/16-2-text-capture-digestion-pipeline.md` — pipeline texte

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Completion Notes List

TBD

### File List

TBD
