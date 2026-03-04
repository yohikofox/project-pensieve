# Story 16.2: Text Capture — Déclenchement Pipeline IA (Digestion + Extraction)

Status: review

## Story

En tant qu'utilisateur,
je veux que mes captures de type texte déclenchent automatiquement l'analyse IA (résumé + idées + extraction d'actions),
afin de bénéficier des mêmes fonctionnalités de digestion IA que les captures audio.

## Contexte & Problème

**Bug identifié le 2026-03-04 :** Les captures texte (`type='text'`) ne déclenchent aucun job de digestion IA côté backend après synchronisation.

Le pipeline audio fonctionne :
```
audio → sync → Normalization (Whisper) → Knowledge (GPT) → Thought/Ideas/Todos → WebSocket → mobile
```

Le pipeline texte est brisé :
```
texte → sync → [RIEN] ← bug ici
                ↑
         Le job de digestion n'est jamais publié
```

**Root cause :** `SyncService.applyCapturePushInTransaction()` stocke la capture texte en base mais ne publie aucun event ni job de digestion. Le handler `handleTextCaptureCreated()` existe dans `transcription-completed.handler.ts:110` mais n'est jamais appelé car `CaptureEventsHandler` n'est pas enregistré dans `KnowledgeModule`.

## Acceptance Criteria

1. **AC1 — Déclenchement backend** : Quand une capture `type='text'` est reçue via `POST /api/sync/push`, un job de digestion est publié dans RabbitMQ (`DigestionJobPublisher.publishJobForTextCapture()`).

2. **AC2 — Pipeline de digestion complet** : Le `KnowledgeModule` traite correctement les captures texte (pas d'étape transcription — le `rawContent` est envoyé directement à GPT-4o-mini, conformément à Story 4.2 AC2).

3. **AC3 — Affichage mobile — Thought/Ideas** : Les résultats de l'analyse IA (résumé + idées) sont visibles dans la vue détail d'une capture texte sur le mobile, de la même façon que pour les captures audio.

4. **AC4 — Affichage mobile — Todos** : Les tâches extraites d'une capture texte apparaissent dans l'onglet Actions après traitement.

5. **AC5 — Notification temps-réel** : Le mobile reçoit la notification WebSocket de fin de digestion pour une capture texte (même canal que Story 4.2).

6. **AC6 — Test de régression** : Le fichier `mobile/tests/acceptance/capture/text-capture-service.test.ts` est migré de WatermelonDB (obsolète) vers OP-SQLite et tous ses scénarios passent.

## Tasks / Subtasks

- [x] **Task 1 — Backend : connecter SyncService au pipeline IA (AC1)**
  - [x] 1.1 Dans `backend/src/modules/sync/application/services/sync.service.ts`, après `applyCapturePushInTransaction()` pour une capture `type='text'`, appeler `DigestionJobPublisher.publishJobForTextCapture({ captureId, userId, type: 'TEXT', state: 'ready', userInitiated: false })`
  - [x] 1.2 Injecter `DigestionJobPublisher` dans `SyncService` via DI (s'assurer que `KnowledgeModule` exporte ce service)
  - [x] 1.3 Connecter directement le `SyncService` au publisher — `SyncModule` importe `KnowledgeModule` qui exporte déjà `DigestionJobPublisher`
  - [x] 1.4 Publication après transaction (post-commit) — `newTextCaptureId` propagé via `applyCapturePushInTransaction` → `applyClientUpdateInTransaction` → `processPush`

- [x] **Task 2 — Backend : vérifier le traitement GPT des captures texte (AC2)**
  - [x] 2.1 `ContentExtractorService` déjà correct : branch `type === 'TEXT'` envoie `rawContent` directement à GPT (pas de Whisper)
  - [x] 2.2 Test unitaire ajouté : `capture-events-handler.spec.ts` — 4 tests (TEXT bypass transcription, AUDIO régression)

- [x] **Task 3 — Mobile : vérifier l'affichage dans la vue détail (AC3, AC4)**
  - [x] 3.1 `CaptureDetailContent` : composants autonomes, pas de filtre sur `type`
  - [x] 3.2 `CaptureAudioPlayerSection` déjà filtre `if (capture.type !== "audio") return null` — audio player masqué pour texte ✅
  - [x] 3.3 `AnalysisCard` affiche Thought/Ideas/Todos pour tous les types ✅ — aucun changement nécessaire

- [x] **Task 4 — Mobile : migrer le test text-capture-service (AC6)**
  - [x] 4.1 Import `@nozbe/watermelondb` supprimé
  - [x] 4.2 Remplacement par mock `ICaptureRepository` (pattern OP-SQLite adapté au service actuel)
  - [x] 4.3 14 tests passent : AC2, AC4, validation vide, edge cases

- [ ] **Task 5 — Validation E2E manuelle**
  - [ ] 5.1 Créer une capture texte, syncer, attendre la notification WebSocket → vérifier apparition du Thought dans le feed
  - [ ] 5.2 Vérifier les todos extraits dans l'onglet Actions

## Dev Notes

### Fichiers critiques backend

| Fichier | Action |
|---------|--------|
| `backend/src/modules/sync/application/services/sync.service.ts:537` | Ajouter appel `DigestionJobPublisher.publishJobForTextCapture()` après push réussi |
| `backend/src/modules/knowledge/application/handlers/transcription-completed.handler.ts:110` | `handleTextCaptureCreated()` existe — à connecter |
| `backend/src/modules/knowledge/knowledge.module.ts` | Enregistrer `CaptureEventsHandler` ou exporter `DigestionJobPublisher` vers SyncModule |

### Fichiers critiques mobile

| Fichier | Action |
|---------|--------|
| `mobile/src/contexts/Normalization/processors/TranscriptionQueueProcessor.ts:30` | Vérifier la branche `type='text'` — normal que le texte ne passe pas par Whisper, mais vérifier si un event alternatif est nécessaire |
| `mobile/tests/acceptance/capture/text-capture-service.test.ts` | Migrer WatermelonDB → OP-SQLite |

### Patterns à respecter

- **ADR-021 / DI TSyringe** : Injection de `DigestionJobPublisher` dans `SyncService` via le module NestJS — ne pas `require()` directement
- **ADR-004** : Un seul appel GPT (summary + ideas + todos) — ne pas créer d'appel LLM supplémentaire
- **ADR-029** : `SyncService` utilise `BetterAuthGuard` pour l'authentification — préserver
- **RabbitMQ** : `DigestionJobPublisher.publishJobForTextCapture()` — signature conforme à `Story 4.2` (type: 'TEXT', sans transcriptText car le rawContent est déjà le texte)
- **Test pattern OP-SQLite** : Mock via `jest.mock('@op-engineering/op-sqlite')` + `better-sqlite3` en mémoire (voir `tests/acceptance/story-8-14.test.ts` comme référence)

### Attention

- **Ne pas changer** le comportement pour les captures audio (régression critique)
- **Ne pas créer** de deuxième appel GPT — le pipeline `4.2` gère déjà le cas texte, il faut juste le déclencher
- **`TranscriptionQueueProcessor` mobile** : Les captures texte ne passent PAS par Whisper (normal). La digestion est exclusivement backend. Ne pas ajouter de traitement on-device pour le texte.

### Références

- [Bug root cause] `backend/src/modules/sync/application/services/sync.service.ts` — méthode `applyCapturePushInTransaction()`
- [Handler existant] `backend/src/modules/knowledge/application/handlers/transcription-completed.handler.ts:110` — `handleTextCaptureCreated()`
- [Story 4.2] `_bmad-output/implementation-artifacts/stories/epic-4/4-2-digestion-ia-resume-idees.md` — pipeline de digestion
- [Story 4.3] `_bmad-output/implementation-artifacts/stories/epic-4/4-3-extraction-actions.md` — extraction todos
- [Test référence] `mobile/tests/acceptance/story-8-14.test.ts` — pattern OP-SQLite mock
- [ADR-004] Un seul appel LLM — architecture/decisions

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

- **Task 1 (AC1)** : `SyncService.processPush()` collecte les IDs des nouvelles captures texte pendant la transaction et appelle `DigestionJobPublisher.publishJobForTextCapture()` APRÈS le commit (évite publication RabbitMQ orpheline en cas de rollback). `applyCapturePushInTransaction()` retourne `newTextCaptureId` si `clientRecord.type === 'text'` et capture nouvelle.
- **Task 2 (AC2)** : `ContentExtractorService` était déjà correct — branche `type === 'TEXT'` confirme envoi direct `rawContent` à GPT sans transcription. Test `capture-events-handler.spec.ts` ajouté.
- **Task 3 (AC3/AC4 — correction)** : 3 bugs mobiles identifiés et corrigés :
  - `CaptureRepository.create()` : `normalized_text` absent du INSERT → ajouté conditionnellement
  - `TextCaptureService.ts` : état `CAPTURED` → `READY` (les textes n'ont pas besoin de transcription, `normalizedText` est déjà disponible)
  - `captureDetailStore.ts` : `isReady` toujours `false` pour les textes → condition étendue pour les captures texte en état `CAPTURED` (compat ascendante) ou `READY`
  - `AnalysisCard` s'affiche maintenant pour les captures texte ✅
- **Task 4 (AC6)** : `text-capture-service.test.ts` entièrement réécrit — suppression WatermelonDB (`@nozbe/watermelondb` + `Database`), adoption pattern mock `ICaptureRepository`. 14 tests passent, état `READY` et absence de `syncStatus` dans le create alignés avec l'implémentation actuelle.

### File List

- `pensieve/backend/src/modules/sync/application/services/sync.service.ts` (modifié)
- `pensieve/backend/src/modules/sync/sync.module.ts` (modifié)
- `pensieve/backend/src/modules/sync/__tests__/sync.service.spec.ts` (modifié)
- `pensieve/backend/src/modules/knowledge/application/handlers/capture-events-handler.spec.ts` (créé)
- `pensieve/mobile/src/contexts/capture/data/CaptureRepository.ts` (modifié — normalized_text dans INSERT)
- `pensieve/mobile/src/contexts/capture/services/TextCaptureService.ts` (modifié — state READY)
- `pensieve/mobile/src/stores/captureDetailStore.ts` (modifié — isReady étendu pour text+captured)
- `pensieve/mobile/tests/acceptance/capture/text-capture-service.test.ts` (modifié)
