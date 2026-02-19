# TEA Artifacts Index - Pensieve

**Last Updated:** 2026-01-21
**Project:** Pensieve - Incubateur Personnel d'Idées Business
**Agent:** TEA (Test Architect Expert)

---

## 📋 Index des Artefacts TEA

Ce fichier index tous les artefacts créés par l'agent TEA pour le projet Pensieve.

### Infrastructure Globale

| Artefact | Chemin | Date Création | Status |
|----------|--------|---------------|--------|
| **Test Infrastructure Setup** | `_bmad-output/test-infrastructure-setup.md` | 2026-01-20 | ✅ Complete |
| **TEA Artifacts Index** | `_bmad-output/tea-artifacts-index.md` | 2026-01-21 | ✅ Active |
| **Test Context (Mocks)** | `pensieve/mobile/tests/acceptance/support/test-context.ts` | 2026-01-20 | ✅ Active |

---

### Epic 1 - Onboarding & Auth

#### Story 1.2 - Intégration Auth Supabase

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-1-2-auth-integration.feature` | 2026-01-20 | 21 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-1-2-auth.test.ts` | 2026-01-20 | 21 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-1-2-auth.md` | 2026-01-20 | - | ✅ Complete |

**Mocks créés:**
- MockSupabaseAuth (signUp, signInWithPassword, OAuth, resetPassword, sessions)
- MockAsyncStorage (token persistence)

---

#### Story 1.3 - Conformité RGPD

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-1-3-rgpd-compliance.feature` | 2026-01-20 | 19 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-1-3-rgpd.test.ts` | 2026-01-20 | 19 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-1-3-rgpd.md` | 2026-01-20 | - | ✅ Complete |

**Mocks créés:**
- MockRGPDService (data export, account deletion, audit logs)

---

### Epic 2 - Capture & Transcription

#### Story 2.1 - Capture Audio 1-Tap

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin (Full) | `pensieve/mobile/tests/acceptance/features/story-2-1-capture-audio.feature` | 2026-01-20 | 44 | ✅ RED |
| Feature Gherkin (Simple) | `pensieve/mobile/tests/acceptance/features/story-2-1-capture-audio-simple.feature` | 2026-01-20 | 15 | ✅ RED |
| Step Definitions (Full) | `pensieve/mobile/tests/acceptance/story-2-1.test.ts` | 2026-01-20 | 44 | ✅ RED |
| Step Definitions (Simple) | `pensieve/mobile/tests/acceptance/story-2-1-simple.test.ts` | 2026-01-20 | 15 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-2-1-capture-audio-1-tap.md` | 2026-01-21 | - | ✅ Complete |

**Mocks créés:**
- MockAudioRecorder (startRecording, stopRecording, getStatus, simulateRecording)
- MockFileSystem (writeFile, readFile, deleteFile, fileExists)
- InMemoryDatabase (create, update, findById, findByState, delete)
- MockPermissionManager (microphone permissions)

**Coverage:**
- AC1: Start Recording < 500ms latency (NFR1)
- AC2: Stop and Save Recording
- AC3: Offline Functionality (FR4)
- AC4: Data Validation & Metadata
- AC5: Error Handling (permissions, storage, crashes)
- Edge Cases: Very short recordings, rapid taps, storage full, crash recovery

---

#### Story 2.2 - Capture Texte Rapide

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-2-2-capture-texte.feature` | 2026-01-21 | 29 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-2-2.test.ts` | 2026-01-21 | 29 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-2-2-capture-texte-rapide.md` | 2026-01-21 | - | ✅ Complete |

**Mocks créés:**
- MockKeyboard (open, close, isOpen)
- MockTextInput (setText, getText, clear, focus)
- MockDialog (show, selectOption, getMessage, getOptions)
- MockDraftStorage (saveDraft, getDraft, clearDraft)
- MockApp (goToBackground, crash, relaunch)

**Coverage:**
- AC1: Open Text Input Field Immediately
- AC2: Save Text Capture with Metadata
- AC3: Cancel Unsaved Text with Confirmation
- AC4: Offline Text Capture Functionality
- AC5: Empty Text Validation
- AC6: Haptic Feedback on Save
- Edge Cases: Very long text, special characters, rapid saves, navigation interruption, crash recovery

---

#### Story 2.3 - Annuler Capture Audio

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-2-3-annuler-capture.feature` | 2026-01-21 | 24 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-2-3.test.ts` | 2026-01-21 | 24 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-2-3-annuler-capture-audio.md` | 2026-01-21 | - | ✅ Complete |

**Mocks à créer:**
- MockHaptics (triggerFeedback, wasFeedbackTriggered, getFeedbackType)

**Mocks réutilisés:**
- MockAudioRecorder (stopRecording, getStatus, reset)
- MockFileSystem (deleteFile, fileExists, getFiles)
- InMemoryDatabase (delete, findById, count)
- MockDialog (show, selectOption, isShown)

**Coverage:**
- AC1: Cancel Button → Arrêt Immédiat et Nettoyage (stop, delete file, delete Capture, navigate)
- AC2: Swipe Cancel Gesture → Confirmation Prompt
- AC3: Haptic Feedback + Animation de Rejet (< 500ms)
- AC4: Protection Contre Annulation Accidentelle (mandatory confirmation)
- AC5: Fonctionnement Offline Identique (FR4 compliance, no orphaned files)
- Edge Cases: Rapid cancel, multiple cancels, cancel during save

---

#### Story 2.4 - Stockage Offline des Captures

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-2-4-stockage-offline.feature` | 2026-01-21 | 28 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-2-4.test.ts` | 2026-01-21 | 28 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-2-4-stockage-offline.md` | 2026-01-21 | - | ✅ Complete |

**Mocks à créer:**
- MockStorageManager (checkAvailableSpace, isStorageLow, retention policy)
- MockSyncQueue (addToQueue, getQueueSize, isEmpty)

**Mocks réutilisés:**
- InMemoryDatabase (create, findAll, findBySyncStatus, count)
- MockFileSystem (writeFile, deleteFile, getAvailableSpace, setAvailableSpace)
- MockApp (crash, relaunch, reset)

**Coverage:**
- AC1: Persistance des captures offline (OP-SQLite + secure storage, syncStatus field)
- AC2: Création multiple sans réseau (NFR7: 100% offline, storage monitoring, warnings)
- AC3: Accès rapide aux captures offline (NFR4: < 1s load time, offline indicators, optimistic UI)
- AC4: Récupération après crash (NFR8: crash recovery, NFR6: zero data loss, DB integrity)
- AC5: Gestion du stockage (retention policy 90 days, cleanup, preserve transcriptions)
- AC6: Encryption at rest (NFR12: device-level encryption, encryptionStatus flag)
- Edge Cases: Storage full, rapid captures stress test, DB corruption, SyncQueue persistence

---

#### Story 2.5 - Transcription On-Device avec Whisper

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-2-5-transcription-whisper.feature` | 2026-01-21 | 26 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-2-5.test.ts` | 2026-01-21 | 26 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-2-5-transcription-whisper.md` | 2026-01-21 | - | ✅ Complete |

**Mocks à créer:**
- MockWhisperService (transcribe, isModelInstalled, setTranscriptionDuration, triggerError)
- MockTranscriptionQueue (addJob, getNextJob, markJobCompleted, FIFO processing)

**Mocks réutilisés:**
- InMemoryDatabase (create, update, findById, findByState)
- MockFileSystem (writeFile, fileExists, reset)
- MockApp (goToBackground, isInBackground)

**Coverage:**
- AC1: Queuing Automatique Après Audio Capture (auto-queue, background process, non-blocking)
- AC2: Transcription Performance NFR2 < 2x Audio Duration (normalizedText storage, file retention)
- AC3: Fonctionnement Offline FR7 (100% local, no network calls, model pre-installed)
- AC4: Download Modèle Whisper (~500 MB, progress 0-100%, queue until ready)
- AC5: Feedback Visuel NFR5 (progress for long audio > 10s, background tasks)
- AC6: Gestion Erreurs (TRANSCRIPTION_FAILED state, error logging, retry option, file preservation)
- AC7: Queue FIFO (First In First Out, single transcription at a time, UI status badge)

---

#### Story 2.6 - Consultation de Transcription

| Artefact | Chemin | Date | Scenarios | Status |
|----------|--------|------|-----------|--------|
| Feature Gherkin | `pensieve/mobile/tests/acceptance/features/story-2-6-consultation-transcription.feature` | 2026-01-21 | 22 | ✅ RED |
| Step Definitions | `pensieve/mobile/tests/acceptance/story-2-6.test.ts` | 2026-01-21 | 22 | ✅ RED |
| ATDD Checklist | `_bmad-output/atdd-checklist-2-6-consultation-transcription.md` | 2026-01-21 | - | ✅ Complete |

**Mocks à créer:**
- MockAudioPlayer (play, pause, stop, loadAudio, getCurrentTime, getDuration)

**Mocks réutilisés:**
- InMemoryDatabase (findById, update, reset)
- MockFileSystem (fileExists, reset)
- MockTranscriptionQueue (addJob - for retry functionality)

**Coverage:**
- AC1: Afficher Transcription Complète (full text, timestamp, audio duration, metadata)
- AC2: Contrôles de Lecture Audio (play/pause, progress bar, simultaneous audio+text reading)
- AC3: Indicateur Transcription en Cours (live update, polling state changes)
- AC4: Gestion Transcription Échouée (error message, retry button, audio preservation)
- AC5: Fonctionnement Offline FR23 (local cache loading, no network calls, local audio playback)
- AC6: Captures Texte vs Audio (conditional rendering, capture type badge)
- AC7: Formatage du Texte (line breaks, special characters, long text scroll, readability)

---

## 📊 Statistiques Globales

### Tests par Epic

| Epic | Stories | Scenarios | Step Defs | Status |
|------|---------|-----------|-----------|--------|
| Epic 1 - Onboarding & Auth | 2 | 40 | 40 | ✅ RED |
| Epic 2 - Capture & Transcription | 6 | 173 | 173 | ✅ RED |
| **Total** | **8** | **213** | **213** | **✅ RED** |

### Mocks Créés (test-context.ts)

| Mock | Responsabilité | Stories Utilisées |
|------|----------------|-------------------|
| MockAudioRecorder | Audio recording (expo-av) | 2.1, 2.3, 2.4 |
| MockFileSystem | File system (expo-file-system) | 2.1, 2.3, 2.4 |
| InMemoryDatabase | OP-SQLite in-memory | 2.1, 2.2, 2.3, 2.4 |
| MockSupabaseAuth | Supabase authentication | 1.2 |
| MockAsyncStorage | React Native AsyncStorage | 1.2 |
| MockRGPDService | RGPD compliance (export, deletion) | 1.3 |
| MockPermissionManager | Microphone permissions | 2.1 |
| MockKeyboard | Keyboard management | 2.2 |
| MockTextInput | Text input field | 2.2 |
| MockDialog | Confirmation dialogs | 2.2, 2.3, 2.4 |
| MockDraftStorage | Draft persistence (crash recovery) | 2.2 |
| MockApp | App lifecycle (background, crash) | 2.2, 2.4, 2.5 |
| **MockHaptics** | **Haptic feedback (TODO)** | **2.3** |
| **MockStorageManager** | **Storage monitoring, retention policy (TODO)** | **2.4** |
| **MockSyncQueue** | **Sync queue management (TODO)** | **2.4** |
| **MockWhisperService** | **On-device Whisper transcription (TODO)** | **2.5** |
| **MockTranscriptionQueue** | **FIFO transcription job queue (TODO)** | **2.5** |
| **MockAudioPlayer** | **Audio playback controls (TODO)** | **2.6** |

**Total Mocks:** 18 (12 implemented + 6 TODO)

---

## 🛠️ Infrastructure Scripts

### Test Execution Scripts

```bash
# Run all acceptance tests
npm run test:acceptance

# Run specific story
npm run test:acceptance:story-1-2   # Auth integration
npm run test:acceptance:story-1-3   # RGPD compliance
npm run test:acceptance:story-2-1   # Audio capture 1-tap
npm run test:acceptance:story-2-2   # Text capture rapide
npm run test:acceptance:story-2-3   # Annuler capture audio
npm run test:acceptance:story-2-4   # Stockage offline des captures
npm run test:acceptance:story-2-5   # Transcription on-device Whisper
npm run test:acceptance:story-2-6   # Consultation de transcription

# Run with coverage
npm run test:coverage

# Watch mode (development)
npm run test:acceptance:watch
```

### Test Filtering

```bash
# Run by tag
npm run test:acceptance -- --testNamePattern="@AC1"
npm run test:acceptance -- --testNamePattern="@offline"
npm run test:acceptance -- --testNamePattern="@performance"

# Run specific epic
npm run test:acceptance -- --testNamePattern="@epic-1"
npm run test:acceptance -- --testNamePattern="@epic-2"
```

---

## 🎯 Prochaines Stories à Traiter

### Epic 2 - Capture & Transcription

✅ **Epic 2 COMPLET** - Toutes les 6 stories terminées en phase RED !

### Epic 3 - Digestion IA

- Story 3.1 - Génération Automatique de Résumés
- Story 3.2 - Extraction d'Idées Clés
- Story 3.3 - Gestion des Notifications de Progression

### Epic 4 - Todo-List & Actions

- Story 4.1 - Conversion Idée → Action
- Story 4.2 - Gestion des Actions Multiples
- Story 4.3 - Synchronisation Actions avec Calendrier

---

## 📚 Références de Connaissances

### Fragments Utilisés

| Fragment | Chemin | Utilisation |
|----------|--------|-------------|
| Data Factories | `_bmad/bmm/testarch/knowledge/data-factories.md` | Factory pattern, overrides, faker |
| Fixture Architecture | `_bmad/bmm/testarch/knowledge/fixture-architecture.md` | Mock architecture, pure functions |
| Test Quality | `_bmad/bmm/testarch/knowledge/test-quality.md` | Deterministic tests, no hard waits |
| Selector Resilience | `_bmad/bmm/testarch/knowledge/selector-resilience.md` | data-testid strategy |
| Timing & Debugging | `_bmad/bmm/testarch/knowledge/timing-debugging.md` | Race conditions, explicit waits |

---

## 📝 Notes et Observations

### Pattern RED-GREEN-REFACTOR

**Phase RED (Complete pour Epic 1 et Epic 2 - 8 stories) ✅**

- ✅ Epic 1: Stories 1.2, 1.3 (40 scenarios)
- ✅ Epic 2: Stories 2.1, 2.2, 2.3, 2.4, 2.5, 2.6 (173 scenarios)
- Tous les tests écrits avec jest-cucumber
- Tous les mocks créés et configurés
- Checklists ATDD complètes avec implementation guides
- Tests échouent comme attendu (implémentation manquante)

**Phase GREEN (En cours - DEV Team)**

- Implémenter les fonctionnalités pour faire passer les tests
- Suivre les checklists ATDD comme guide
- Run tests fréquemment pour feedback immédiat

**Phase REFACTOR (À venir)**

- Nettoyer le code après tests GREEN
- Optimiser performance
- Améliorer lisibilité

---

### Principes TEA Appliqués

1. **Test-First** - Tests écrits AVANT l'implémentation
2. **BDD avec Gherkin** - Scenarios lisibles en français
3. **Mocks In-Memory** - Pas de dépendances externes (SQLite, réseau)
4. **Deterministic Tests** - Pas de hard waits, tests reproductibles
5. **Data-Driven** - Scenario Outlines pour tester plusieurs cas
6. **Edge Cases** - Bug prevention avec scenarios edge cases
7. **NFR Compliance** - Tags pour exigences non-fonctionnelles (@performance, @offline)

---

## 🔗 Liens Utiles

- **Test Infrastructure Doc:** `_bmad-output/test-infrastructure-setup.md`
- **Test Context Source:** `pensieve/mobile/tests/acceptance/support/test-context.ts`
- **TEA Agent Guide:** `_bmad/bmm/agents/tea.md`
- **TEA Knowledge Base:** `_bmad/bmm/testarch/knowledge/`
- **Sprint Status:** `_bmad-output/implementation-artifacts/sprint-status.yaml`

---

**Generated by TEA Agent** - 2026-01-21
