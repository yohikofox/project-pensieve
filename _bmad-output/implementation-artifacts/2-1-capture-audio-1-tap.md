# Story 2.1: Capture Audio 1-Tap

Status: ready-for-dev

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
**And** a Capture entity is created in WatermelonDB with status "recording"
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

- [ ] **Task 1: Setup Capture Context Mobile Infrastructure** (AC: 1, 2, 3, 5)
  - [ ] Subtask 1.1: Create Capture aggregate model in WatermelonDB
    - Define schema with fields: id, type='audio', state, rawContent (file path), normalizedText, capturedAt, location, tags, syncStatus
    - Implement WatermelonDB model with proper decorators and relationships
    - Add migration script for Capture table
  - [ ] Subtask 1.2: Configure expo-av audio recording permissions
    - Add microphone permission to app.json/app.config.js
    - Implement permission request flow with user-friendly messaging
    - Handle permission denied state gracefully
  - [ ] Subtask 1.3: Implement RecordingService with expo-av
    - Create service to handle audio recording lifecycle
    - Configure recording options (format, quality, channels)
    - Implement start/stop recording with state management
    - Save audio files to secure device storage with proper naming
  - [ ] Subtask 1.4: Create React component for record button UI
    - Implement pulsing red indicator during recording
    - Add haptic feedback using expo-haptics (iOS/Android)
    - Ensure < 500ms tap-to-record latency (NFR1)
    - Display recording duration timer

- [ ] **Task 2: Implement Offline-First Capture Persistence** (AC: 3, 4)
  - [ ] Subtask 2.1: Implement auto-save during recording
    - Stream audio data to temporary file during recording
    - Atomic file operations to prevent corruption
    - Handle low storage scenarios gracefully
  - [ ] Subtask 2.2: Implement crash recovery mechanism
    - Detect incomplete recordings on app launch
    - Attempt recovery of partial audio files
    - Store recovery metadata in WatermelonDB
    - Notify user of recovered captures
  - [ ] Subtask 2.3: Mark Capture entities for sync
    - Add syncStatus field (pending/synced)
    - Implement offline queue for pending captures
    - Ensure WatermelonDB sync protocol compatibility

- [ ] **Task 3: Implement Audio File Storage Management** (AC: 2)
  - [ ] Subtask 3.1: Define audio file naming convention
    - Use pattern: `capture_{userId}_{timestamp}_{uuid}.m4a`
    - Store in secure directory (FileSystem.documentDirectory)
  - [ ] Subtask 3.2: Store metadata with Capture entity
    - Record duration, file size, timestamp
    - Store file path reference in Capture.rawContent
    - Track recording quality/format metadata

- [ ] **Task 4: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 4.1: Unit tests for RecordingService
    - Test recording start/stop lifecycle
    - Test permission handling
    - Test file creation and naming
    - Test crash recovery logic
  - [ ] Subtask 4.2: Unit tests for Capture model
    - Test entity creation with correct schema
    - Test state transitions (recording → captured)
    - Test offline sync status tracking
  - [ ] Subtask 4.3: Integration tests for capture flow
    - Test end-to-end recording from button tap to save
    - Test offline capture functionality
    - Test permission request flow
    - Mock crash scenarios and verify recovery
  - [ ] Subtask 4.4: Performance tests
    - Verify < 500ms latency from tap to recording start (NFR1)
    - Test with various audio durations (30s, 2min, 5min)
    - Verify memory usage during long recordings

## Dev Notes

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
- WatermelonDB (offline-first avec sync protocol)
- expo-av pour audio recording
- expo-haptics pour feedback tactile
- expo-file-system pour stockage sécurisé

**Storage:**
- WatermelonDB SQLite local database
- Secure file system pour audio files
- Sync protocol pour synchronisation cloud future

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
│   │   └── Capture/  # Bounded Context
│   │       ├── domain/
│   │       │   ├── Capture.model.ts  # WatermelonDB model
│   │       │   └── Capture.schema.ts # DB schema
│   │       ├── services/
│   │       │   ├── RecordingService.ts  # expo-av wrapper
│   │       │   └── CrashRecoveryService.ts
│   │       └── ui/
│   │           └── RecordButton.tsx
│   └── ...
```

### Previous Story Intelligence

**From Epic 1 (Foundation):**
- Project structure établie (React Native + Expo + NestJS)
- WatermelonDB déjà configuré
- Identity Context implémenté (auth disponible)
- Docker Compose homelab configuré
- PostgreSQL + RabbitMQ + Redis opérationnels

**Key Learnings:**
- TypeScript strict mode activé partout
- Jest + @testing-library/react-native pour tests mobile
- DDD folder structure suivie rigoureusement

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.1]
- [Source: _bmad-output/planning-artifacts/architecture.md#Capture Context]
- [Source: _bmad-output/planning-artifacts/architecture.md#Technology Stack - Mobile]
- [Source: _bmad-output/planning-artifacts/architecture.md#DDD Aggregates - Capture]

### Dependencies & Libraries

**Required npm packages:**
- `expo-av` (~49.0.0): Audio recording et playback
- `expo-haptics` (~12.8.0): Feedback haptique
- `expo-file-system` (~16.0.0): File storage management
- `@nozbe/watermelondb` (~0.27.0): Offline-first database

**Already installed (from Epic 1):**
- `react-native` + `expo`
- TypeScript tooling
- Jest testing framework

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->
