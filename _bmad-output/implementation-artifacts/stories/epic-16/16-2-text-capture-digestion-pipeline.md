# Story 16.2: Text Capture — Déclenchement Pipeline IA (Digestion + Extraction)

Status: ready-for-dev

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

- [ ] **Task 1 — Backend : connecter SyncService au pipeline IA (AC1)**
  - [ ] 1.1 Dans `backend/src/modules/sync/application/services/sync.service.ts`, après `applyCapturePushInTransaction()` pour une capture `type='text'`, appeler `DigestionJobPublisher.publishJobForTextCapture({ captureId, userId, type: 'TEXT', state: 'ready', userInitiated: false })`
  - [ ] 1.2 Injecter `DigestionJobPublisher` dans `SyncService` via DI (s'assurer que `KnowledgeModule` exporte ce service)
  - [ ] 1.3 Enregistrer `CaptureEventsHandler` dans `backend/src/modules/knowledge/knowledge.module.ts` OU connecter directement le `SyncService` au publisher (sans passer par l'event bus si celui-ci n'est pas encore branché)
  - [ ] 1.4 Vérifier que `handleTextCaptureCreated()` dans `transcription-completed.handler.ts:110` est bien invoqué

- [ ] **Task 2 — Backend : vérifier le traitement GPT des captures texte (AC2)**
  - [ ] 2.1 Confirmer que `ContentExtractorService` ou équivalent envoie `rawContent` directement à GPT pour `type='text'` (sans passer par le step de transcription)
  - [ ] 2.2 Ajouter un test unitaire backend pour le handler texte

- [ ] **Task 3 — Mobile : vérifier l'affichage dans la vue détail (AC3, AC4)**
  - [ ] 3.1 Identifier si la vue détail d'une capture (`CaptureDetailScreen` ou `CaptureDetailContent`) conditionne l'affichage des sections Thought/Ideas/Todos au type de capture
  - [ ] 3.2 Si conditionnel sur `type='audio'`, étendre pour afficher aussi pour `type='text'`
  - [ ] 3.3 Vérifier que le WebSocket listener traite bien les notifications pour les captures texte (AC5)

- [ ] **Task 4 — Mobile : migrer le test text-capture-service (AC6)**
  - [ ] 4.1 Supprimer l'import `@nozbe/watermelondb` dans `mobile/tests/acceptance/capture/text-capture-service.test.ts`
  - [ ] 4.2 Remplacer par le mock OP-SQLite (pattern de `tests/acceptance/support/test-context.ts`)
  - [ ] 4.3 S'assurer que tous les scénarios du fichier `.feature` associé passent

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

### File List
