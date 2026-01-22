# Story 2.1: Capture Audio 1-Tap

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **to record an audio thought with a single tap from the main screen**,
so that **I can quickly capture my ideas without friction, even without network connectivity**.

## Acceptance Criteria

### AC1: Start Recording with < 500ms Latency
**Given** I am on the main screen of the app
**When** I tap the record button
**Then** audio recording starts within 500ms (NFR1 compliance)
**And** visual feedback is displayed (pulsing red indicator)
**And** haptic feedback is triggered on iOS/Android
**And** a Capture entity is created in OP-SQLite database with status "recording"
**And** audio data is streamed to local storage

### AC2: Stop and Save Recording
**Given** I am recording audio
**When** I tap the stop button
**Then** the recording stops immediately
**And** the audio file is saved to device storage
**And** the Capture entity is updated with status "captured" and file path
**And** the audio file metadata (duration, size, timestamp) is stored

### AC3: Offline Functionality
**Given** I have no network connectivity
**When** I start and complete an audio recording
**Then** the capture works identically to online mode (FR4, NFR7 compliance)
**And** the Capture entity is marked for future sync
**And** no error is shown to the user

### AC4: Crash Recovery
**Given** the app crashes during recording
**When** I reopen the app
**Then** the partial recording is recovered if possible (NFR8 compliance)
**And** I receive a notification about the recovered capture

### AC5: Microphone Permission Handling
**Given** microphone permission is not granted
**When** I attempt to record
**Then** I am prompted to grant microphone access
**And** recording only starts after permission is granted

## Tasks / Subtasks

- [x] **Task 1: Setup Capture Context Mobile Infrastructure** (AC: 1, 2, 3, 5)
  - [x] Subtask 1.1: Create Capture aggregate model with OP-SQLite
    - ✅ Capture model implemented at `mobile/src/contexts/capture/domain/Capture.model.ts`
    - ✅ Uses TypeScript interfaces + OP-SQLite repository pattern (ADR-018 decision)
    - ✅ ICaptureRepository interface defines persistence contract
    - ✅ MockCaptureRepository for testing, SQLiteCaptureRepository for production
    - ⚠️ **ARCHITECTURE CHANGE** - OP-SQLite remplace WatermelonDB (voir [ADR-018](../../planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md))
    - Raison: WatermelonDB incompatible JSI + 150+ issues GitHub + non maintenu depuis 18 mois
  - [x] Subtask 1.2: Configure expo-av audio recording permissions
    - ✅ Microphone permissions configured in app.json
    - ✅ iOS: NSMicrophoneUsageDescription set
    - ✅ Android: RECORD_AUDIO permission added
    - ✅ PermissionService implements permission request flow with user-friendly messaging
    - ✅ Permission denied state handled gracefully
  - [x] Subtask 1.3: Implement RecordingService with expo-av
    - ✅ RecordingService implemented at `mobile/src/contexts/capture/services/RecordingService.ts`
    - ✅ Uses expo-audio SDK 54+ (expo-av deprecated)
    - ✅ Configured for m4a format (Android/iOS/Web)
    - ✅ Implements start/stop recording with state management
    - ✅ Saves audio files with proper naming convention
    - ✅ Has unit tests at `__tests__/RecordingService.test.ts`
  - [x] Subtask 1.4: Create React component for record button UI
    - ✅ RecordButton.tsx created at `mobile/src/contexts/capture/ui/RecordButton.tsx`
    - ✅ Pulsing red indicator during recording (Animated API with loop + sequence)
    - ✅ Haptic feedback (expo-haptics integrated, Medium impact on tap)
    - ✅ Recording duration timer displayed (MM:SS format, updates every second)
    - ✅ Visual states: idle (red circle) vs recording (red circle with white dot + pulsing border)
    - ✅ Integration with RecordingService via TSyringe container
    - ✅ Callbacks: onRecordingStart, onRecordingStop, onError
    - ✅ Unit tests: 7/7 tests passing
    - ✅ testID="record-button" for E2E testing

- [x] **Task 2: Implement Offline-First Capture Persistence** (AC: 3, 4)
  - [x] Subtask 2.1: Implement auto-save during recording
    - ✅ Auto-save logic in RecordingService
    - ✅ Streams audio data to file during recording
    - ✅ Atomic file operations via expo-file-system
    - ⚠️ Low storage handling not explicitly implemented
  - [x] Subtask 2.2: Implement crash recovery mechanism
    - ✅ CrashRecoveryService implemented at `mobile/src/contexts/capture/services/CrashRecoveryService.ts`
    - ✅ Detects incomplete recordings on app launch
    - ✅ Attempts recovery of partial audio files
    - ✅ Has unit tests at `__tests__/CrashRecoveryService.test.ts`
    - ✅ User notification implemented (AC4: "I receive a notification about the recovered capture")
      - Notification utility at `mobile/src/shared/utils/notificationUtils.ts`
      - Integrated in App.tsx on launch via useEffect
      - Has tests at `mobile/src/shared/utils/__tests__/notificationUtils.test.ts` (8/8 passing)
  - [x] Subtask 2.3: Mark Capture entities for sync
    - ✅ OfflineSyncService implemented at `mobile/src/contexts/capture/services/OfflineSyncService.ts`
    - ✅ Implements syncStatus field (pending/synced)
    - ✅ Implements offline queue for pending captures
    - ✅ Has unit tests at `__tests__/OfflineSyncService.test.ts`
    - ⚠️ Sync protocol sera implémenté manuellement Epic 6 (OP-SQLite ne fournit pas sync built-in, voir ADR-018)

- [x] **Task 3: Implement Audio File Storage Management** (AC: 2)
  - [x] Subtask 3.1: Define audio file naming convention
    - ✅ Pattern implemented: `capture_{userId}_{timestamp}_{uuid}.m4a`
    - ✅ Stored in secure directory: FileSystem.documentDirectory + audio/
    - ✅ Implemented in RecordingService.generateFilePath()
  - [x] Subtask 3.2: Store metadata with Capture entity
    - ✅ FileStorageService records duration, file size, timestamp
    - ✅ File path stored in Capture.rawContent
    - ✅ Tracking quality/format metadata via FileMetadata interface
    - ✅ Has unit tests at `__tests__/FileStorageService.test.ts`

- [x] **Task 4: Write Comprehensive Tests** (AC: All)
  - [x] Subtask 4.1: Unit tests for RecordingService
    - ✅ Tests at `mobile/src/contexts/capture/services/__tests__/RecordingService.test.ts`
    - ✅ Tests recording start/stop lifecycle
    - ✅ Tests permission handling
    - ✅ Tests file creation and naming
  - [x] Subtask 4.2: Unit tests for Capture model
    - ✅ Tests at `mobile/src/contexts/capture/domain/__tests__/Capture.model.test.ts`
    - ✅ Tests entity creation with correct schema
    - ✅ Tests state transitions (recording → captured)
    - ✅ Tests offline sync status tracking
  - [x] Subtask 4.3: Integration tests for capture flow
    - ✅ Integration tests at `mobile/src/contexts/capture/__tests__/capture-integration.test.ts`
    - ✅ Acceptance tests at `tests/acceptance/story-2-1-simple.test.ts` (3 tests passing)
    - ✅ Tests end-to-end recording from tap to save
    - ⚠️ Offline network mocking not explicitly tested (issue noted in code review)
  - [x] Subtask 4.4: Performance tests
    - ✅ Performance tests at `mobile/src/contexts/capture/__tests__/capture-performance.test.ts`
    - ✅ Verifies < 500ms latency from tap to recording start (NFR1)
    - ✅ Tests with various audio durations
    - ✅ Memory usage verification included

### Review Follow-ups (AI Code Review - 2026-01-22)

**Code Review Agent:** Adversarial Senior Developer Review
**Review Date:** 2026-01-22
**Issues Found:** 4 High, 5 Medium

#### 🔴 High Priority Issues

- [x] **[AI-Review][HIGH]** Story File Tasks Not Checked - Toutes les tasks marquées [ ] alors que IoC/DI est implémenté. Synchroniser story file avec réalité. [2-1-capture-audio-1-tap.md:53-115]
  - ✅ **RÉSOLU** (2026-01-22) - Tasks synchronisées avec checkmarks corrects
  - Tasks 2, 3, 4 marquées [x] (complètes)
  - Task 1 partiellement complète : Subtasks 1.2, 1.3 [x] / Subtasks 1.1, 1.4 [ ]

- [x] **[AI-Review][HIGH]** UI Components Completely Missing - AC1 demande RecordButton component, haptic feedback, pulsing red indicator. Aucun n'existe. Implémenter Subtask 1.4 complètement. [mobile/src/contexts/capture/ui/RecordButton.tsx:NOT_CREATED]
  - ✅ **RÉSOLU** (2026-01-22) - RecordButton.tsx créé avec tous les features requis
  - ✅ expo-haptics installé et intégré (haptic feedback sur tap iOS/Android)
  - ✅ Pulsing red indicator implémenté avec Animated API
  - ✅ Recording timer affiché (format MM:SS)
  - ✅ Tests unitaires complets (7/7 pass): Initial state, haptic feedback, service integration, callbacks
  - ✅ testID ajouté pour faciliter les tests E2E

- [x] **[AI-Review][HIGH]** WatermelonDB Not Used - Story exige WatermelonDB schema/model/migrations mais utilise in-memory repository. Implémenter Subtask 1.1 ou justifier choix architecture. [mobile/src/contexts/capture/domain/Capture.model.ts:NOT_CREATED]
  - ✅ **RÉSOLU** (2026-01-22) - Architecture changée après Story 2.1 écrite
  - ✅ [ADR-018](../../planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md) documente migration WatermelonDB → OP-SQLite
  - ✅ Raisons techniques bloquantes: WatermelonDB JSI incompatible + 150+ issues + non maintenu 18+ mois
  - ✅ OP-SQLite score 9.2/10 vs WatermelonDB 3.5/10 (performance +300%, bundle -83%)
  - ✅ Repository pattern implémenté: ICaptureRepository + MockCaptureRepository (tests) + SQLiteCaptureRepository (prod)
  - ✅ Subtask 1.1 mise à jour pour refléter OP-SQLite
  - ⚠️ Trade-off accepté: Sync protocol manuel Epic 6 (vs sync built-in WatermelonDB)

- [x] **[AI-Review][HIGH]** Crash Recovery Notification Missing - AC4 exige "notification about recovered capture" mais CrashRecoveryService ne notifie pas l'utilisateur. Ajouter notification UI. [mobile/src/contexts/capture/services/CrashRecoveryService.ts:51]
  - ✅ **RÉSOLU** (2026-01-22) - Notification utilisateur implémentée
  - Notification utility créée: `mobile/src/shared/utils/notificationUtils.ts`
  - Fonction `showCrashRecoveryNotification()` avec Alert natif React Native (MVP approach)
  - Intégration App.tsx: useEffect appelle crash recovery au lancement
  - Tests: 8/8 passing (`mobile/src/shared/utils/__tests__/notificationUtils.test.ts`)
  - Gère 3 scénarios: succès complet, échec complet, récupération partielle
  - Messages en français avec gestion singulier/pluriel correcte

#### 🟡 Medium Priority Issues

- [x] **[AI-Review][MEDIUM]** Performance Tests Absent - AC1 exige vérification < 500ms latency mais aucun test perf. Implémenter Subtask 4.4 complètement. [mobile/tests/acceptance/performance/:NOT_CREATED]
  - ✅ **RÉSOLU** (2026-01-22) - Tests existent à `mobile/src/contexts/capture/__tests__/capture-performance.test.ts`
  - Tests vérifient latency < 500ms, durées variées (30s, 2min, 5min), memory usage

- [x] **[AI-Review][MEDIUM]** Offline Tests Manquants - AC3 demande test offline avec network mocked mais non implémenté. Compléter Subtask 4.3. [mobile/tests/acceptance/story-2-1-simple.test.ts:NO_OFFLINE_TEST]
  - ✅ **RÉSOLU** (2026-01-22) - Test offline ajouté dans Gherkin et implémenté
  - Scénario Gherkin "@AC3 @offline" créé: `tests/acceptance/features/story-2-1-capture-audio-simple.feature:26-35`
  - Test implementation: `tests/acceptance/story-2-1-simple.test.ts:110-153`
  - Vérifie: capture fonctionne identiquement offline (AC3), syncStatus = "pending"
  - Tests d'acceptance: 4/4 passing (latency, save, offline, permissions)

- [x] **[AI-Review][MEDIUM]** File List Incomplet - Dev Agent Record manque 4 fichiers créés: tokens.ts, container.ts, test-container.ts, MockCaptureRepository.ts. Mettre à jour documentation. [story-2-1-dev-agent-record.md:139-175]
  - ✅ **RÉSOLU** (2026-01-22) - File List mise à jour avec tous les fichiers créés
  - Ajouté 4 fichiers manquants: tokens.ts, container.ts, test-container.ts, MockCaptureRepository.ts
  - Ajouté aussi: RecordButton.tsx, RecordButton.test.tsx, notificationUtils.ts, notificationUtils.test.ts
  - Total: 19 created, 4 modified (incluant tests offline et notifications)
  - File List complète: `story-2-1-dev-agent-record.md:154-183`

- [x] **[AI-Review][MEDIUM]** Story Status Incohérent - Story file dit "ready-for-dev" mais sprint-status.yaml dit "review". Devrait être "in-progress" vu manques UI/Tests. [2-1-capture-audio-1-tap.md:3]
  - ✅ **RÉSOLU** (2026-01-22) - Status changé à "in-progress" dans story file ET sprint-status.yaml
  - ✅ UI implémentée (RecordButton.tsx)
  - ✅ OP-SQLite utilisé (WatermelonDB remplacé par ADR-018)
  - Cohérence établie

- [x] **[AI-Review][MEDIUM]** Package Dependencies Not Documented - package.json/package-lock.json modifiés (tsyringe install) mais non listés dans Dev Agent Record. Documenter changements. [mobile/package.json:MODIFIED]
  - ✅ **RÉSOLU** (2026-01-22) - Section "Package Dependencies" ajoutée au Dev Agent Record
  - Documenté: expo-haptics (^15.0.8), tsyringe (^4.10.0), reflect-metadata (^0.2.2)
  - Documenté devDep: @babel/plugin-proposal-decorators (^7.28.6)
  - Documenté dépendances existantes utilisées: expo-audio, expo-file-system, async-storage, op-sqlite
  - Documentation complète: `story-2-1-dev-agent-record.md:185-214`

## Dev Notes

⚠️ **ARCHITECTURE CHANGE NOTICE** (2026-01-22)

Cette story a été écrite avec WatermelonDB comme database layer. Après implémentation, [ADR-018](../../planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md) a documenté la migration vers **OP-SQLite** pour raisons techniques critiques:
- WatermelonDB incompatible avec JSI (Expo SDK 54+)
- 150+ issues GitHub, non maintenu depuis 18+ mois
- OP-SQLite: 9.2/10 score (vs 3.5/10), performance +300%, bundle -83%

**Impact**: Repository pattern utilisé (ICaptureRepository) pour abstraire la persistence. Sync protocol sera implémenté manuellement Epic 6.

---

### Architecture Context

**Bounded Context:** Capture Context (Supporting Domain)
**Aggregate:** `Capture` (polymorphic: audio | text | image | url)
**Domain Events:** `CaptureRecorded`, `CaptureNormalized`

**DDD Model (from architecture.md):**
```typescript
Capture {
  id: UUID
  type: 'audio' | 'text' | 'image' | 'url'
  state: 'captured' | 'processing' | 'ready' | 'failed'
  projectId?: UUID  // Null for orphaned captures
  rawContent: AudioFile | string | ImageFile | URL
  normalizedText?: string  // Set after transcription
  capturedAt: DateTime
  location?: GeoPoint
  tags?: string[]
  syncStatus: 'pending' | 'synced'
}
```

### Technical Stack (from architecture.md)

**Mobile:**
- React Native + Expo (custom dev client for Whisper)
- TypeScript strict mode
- OP-SQLite (offline-first, JSI-native, 9.2/10 score - voir [ADR-018](../../planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md))
- expo-audio pour audio recording
- expo-haptics pour feedback tactile
- expo-file-system pour stockage sécurisé
- TSyringe pour Dependency Injection

**Storage:**
- OP-SQLite local database (JSI-native, 12,000 ops/sec, 380KB bundle)
- Secure file system pour audio files
- Custom sync protocol Epic 6 (OP-SQLite ne fournit pas sync built-in)

### Performance Requirements

**NFR1:** Capture audio < 500ms après tap
**NFR6:** 0 capture perdue, jamais (tolérance zéro)
**NFR7:** Disponibilité capture offline = 100%
**NFR8:** Récupération après crash automatique

### UX Requirements (Liquid Glass Design System)

- Animations fluides 60fps obligatoires
- Feedback haptique iOS/Android sur tap record
- Pulsing red indicator visuel pendant enregistrement
- Micro-animations pour germination (pas cette story, mais préparer)
- SwiftUI-like animations iOS / Material Design 3 Android

### Security Considerations

**NFR12:** Chiffrement au repos (device + cloud future)
- Audio files stockés dans secure storage
- Metadata chiffrés avec encryption device-level
- Aucune donnée audio ne transite par tiers non contrôlés

### Testing Standards

- Unit tests pour RecordingService et Capture model
- Integration tests pour flow complet tap→save
- Performance tests pour vérifier < 500ms latency
- Crash recovery simulation tests
- Offline mode tests (network mocked off)

### File Structure (ADR-007: From Scratch)

```
apps/mobile/
├── src/
│   ├── contexts/
│   │   └── capture/  # Bounded Context
│   │       ├── domain/
│   │       │   ├── Capture.model.ts        # TypeScript interface
│   │       │   ├── ICaptureRepository.ts   # Repository contract
│   │       │   └── Result.ts               # Result pattern types
│   │       ├── data/
│   │       │   ├── MockCaptureRepository.ts   # In-memory pour tests
│   │       │   └── SQLiteCaptureRepository.ts # OP-SQLite impl (Epic 6)
│   │       ├── services/
│   │       │   ├── RecordingService.ts        # expo-audio wrapper
│   │       │   ├── CrashRecoveryService.ts
│   │       │   └── OfflineSyncService.ts
│   │       └── ui/
│   │           └── RecordButton.tsx
│   ├── infrastructure/
│   │   └── di/
│   │       ├── tokens.ts    # TSyringe DI tokens
│   │       └── container.ts # IoC container setup
│   └── ...
```

### Previous Story Intelligence

**From Epic 1 (Foundation):**
- Project structure établie (React Native + Expo + NestJS)
- ⚠️ **ARCHITECTURE CHANGE**: WatermelonDB → OP-SQLite (ADR-018 - JSI incompatibility)
- Identity Context implémenté (auth disponible)
- Docker Compose homelab configuré
- PostgreSQL + RabbitMQ + Redis opérationnels

**Key Learnings:**
- TypeScript strict mode activé partout
- Jest + @testing-library/react-native pour tests mobile
- DDD folder structure suivie rigoureusement
- TSyringe DI container implémenté (ADR-017)
- Repository pattern pour abstraire persistence layer

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.1]
- [Source: _bmad-output/planning-artifacts/architecture.md#Capture Context]
- [Source: _bmad-output/planning-artifacts/architecture.md#Technology Stack - Mobile]
- [Source: _bmad-output/planning-artifacts/architecture.md#DDD Aggregates - Capture]

### Dependencies & Libraries

**Required npm packages:**
- `expo-audio` (~1.1.1): Audio recording et playback (remplace expo-av deprecated)
- `expo-haptics` (~12.8.0): Feedback haptique iOS/Android
- `expo-file-system` (~16.0.0): File storage management
- `@op-engineering/op-sqlite` (~15.2.3): Offline-first database (JSI-native, voir ADR-018)
- `tsyringe` (~4.8.0): Dependency Injection container (ADR-017)
- `reflect-metadata` (~0.2.0): Required by TSyringe

**Already installed (from Epic 1):**
- `react-native` + `expo`
- TypeScript tooling
- Jest testing framework
- @testing-library/react-native

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->
