# Story 2.5: Transcription On-Device avec Whisper

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **my audio captures to be automatically transcribed locally on my device**,
so that **I can read my thoughts without relying on cloud services** (confidentialité audio).

## Acceptance Criteria

### AC1: Automatic Transcription Trigger Post-Capture
**Given** I have completed an audio recording
**When** the recording is saved
**Then** transcription automatically begins in the background (FR6)
**And** the Capture entity state transitions to "processing"
**And** a progress indicator shows transcription is in progress

### AC2: 100% Local On-Device Transcription
**Given** transcription is initiated
**When** the audio file is processed
**Then** Whisper model runs entirely on-device (no network calls) (FR7)
**And** transcription works identically offline and online
**And** no audio data leaves the device

### AC3: Transcription Performance within 2x Audio Duration
**Given** an audio capture of X seconds
**When** transcription completes
**Then** transcription time ≤ 2x audio duration (NFR2 compliance)
**And** device remains responsive during transcription (background processing)

### AC4: Transcription Result Stored in Capture Entity
**Given** transcription completes successfully
**When** the result is processed
**Then** the Capture entity normalizedText field is populated with transcription
**And** the Capture state transitions to "ready"
**And** the user is notified of completion (if app in background)

### AC5: Transcription Error Handling
**Given** transcription fails (model error, corrupted audio, etc.)
**When** the error is detected
**Then** the Capture entity state transitions to "failed"
**And** the user is notified with a retry option
**And** the original audio file is preserved

### AC6: Whisper Model Download on First Launch
**Given** the app is launched for the first time
**When** the user completes onboarding
**Then** the Whisper model (~500MB) is downloaded (NFR3 compliance)
**And** download progress is shown to the user
**And** transcription features are disabled until download completes

## Tasks / Subtasks

- [ ] **Task 1: Integrate Whisper.rn Native Module** (AC: 2, 6)
  - [ ] Subtask 1.1: Research Whisper.rn library for React Native
    - Evaluate `whisper.rn` package (or equivalent)
    - Verify iOS/Android support
    - Check model size and performance benchmarks
  - [ ] Subtask 1.2: Configure Expo custom dev client
    - Whisper requires native modules (not available in Expo Go)
    - Generate custom dev client with `expo prebuild`
    - Add native dependencies to app.json/app.config.js
  - [ ] Subtask 1.3: Install and configure Whisper.rn
    - Install whisper.rn via npm
    - Link native modules (iOS/Android)
    - Configure build settings (increase heap size for model loading)
  - [ ] Subtask 1.4: Implement model download service
    - Download Whisper model on first launch (~500MB)
    - Store model in secure app directory
    - Show progress bar during download
    - Handle download failures with retry logic

- [ ] **Task 2: Create TranscriptionService** (AC: 1, 3, 4, 5)
  - [ ] Subtask 2.1: Design TranscriptionService architecture
    - Create service to manage transcription lifecycle
    - Queue-based processing (FIFO for pending captures)
    - Background processing (don't block UI)
  - [ ] Subtask 2.2: Implement transcribe() method
    - Load Whisper model (once, cached in memory)
    - Pass audio file path to Whisper.rn
    - Configure Whisper params (language, task=transcribe)
    - Return transcription text
  - [ ] Subtask 2.3: Handle transcription errors
    - Catch model loading errors
    - Catch audio format errors
    - Catch timeout errors (if > 2x duration)
    - Set Capture state to "failed"
    - Preserve original audio file
  - [ ] Subtask 2.4: Implement performance monitoring
    - Track transcription duration vs. audio duration
    - Log if exceeds 2x threshold (NFR2 violation)
    - Monitor device memory usage during transcription

- [ ] **Task 3: Implement Background Transcription Queue** (AC: 1, 3)
  - [ ] Subtask 3.1: Create TranscriptionQueueService
    - Maintain FIFO queue of pending transcriptions
    - Persist queue state to survive app restarts
    - Process one transcription at a time (avoid memory issues)
  - [ ] Subtask 3.2: Automatically enqueue audio captures
    - Listen for CaptureRecorded events
    - Enqueue audio captures (type='audio', state='captured')
    - Skip text captures (no transcription needed)
  - [ ] Subtask 3.3: Implement background processing
    - Process queue in background thread/worker
    - Update Capture state: "captured" → "processing" → "ready"
    - Handle app backgrounding (pause/resume queue)

- [ ] **Task 4: Update Capture Entity for Transcription** (AC: 4)
  - [ ] Subtask 4.1: Verify Capture schema supports transcription
    - Ensure normalizedText field exists (already in schema from architecture)
    - Add transcriptionStatus field ('pending' | 'processing' | 'completed' | 'failed')
    - Add transcriptionError field for error messages
  - [ ] Subtask 4.2: Update Capture state transitions
    - Add "processing" state to Capture state machine
    - Transition: "captured" → "processing" → "ready"
    - Failed path: "processing" → "failed"
  - [ ] Subtask 4.3: Save transcription result
    - Populate Capture.normalizedText with Whisper output
    - Set transcriptionStatus = 'completed'
    - Update Capture state to "ready"

- [ ] **Task 5: Add Transcription Progress UI** (AC: 1, 4, 6)
  - [ ] Subtask 5.1: Show progress indicator on captures
    - Display spinner/progress bar on captures in "processing" state
    - Show "Transcribing..." text
    - Update in real-time as transcription progresses
  - [ ] Subtask 5.2: Show Whisper model download progress
    - Display progress bar during initial model download
    - Show download size and speed
    - Block transcription features until complete
    - Provide "Download Later" option (non-blocking onboarding)
  - [ ] Subtask 5.3: Implement transcription completion notification
    - Send local notification when transcription completes (if app backgrounded)
    - Notification text: "Your thought is ready to read"
    - Tapping notification opens the capture detail view

- [ ] **Task 6: Implement Retry Logic for Failed Transcriptions** (AC: 5)
  - [ ] Subtask 6.1: Show retry button on failed captures
    - Display "Transcription failed" badge
    - Show "Retry" button in capture detail view
    - Store error message for debugging
  - [ ] Subtask 6.2: Implement manual retry
    - Re-enqueue failed capture for transcription
    - Reset Capture state: "failed" → "captured"
    - Clear previous error message
  - [ ] Subtask 6.3: Implement automatic retry with exponential backoff
    - Retry failed transcriptions 3 times
    - Backoff: 5s, 30s, 5min
    - Stop after 3 failures, require manual retry

- [ ] **Task 7: Optimize Whisper Performance** (AC: 3)
  - [ ] Subtask 7.1: Configure Whisper model size
    - Use Whisper "tiny" or "base" model for speed
    - Trade-off: accuracy vs. performance
    - Target: < 2x audio duration (NFR2)
  - [ ] Subtask 7.2: Optimize audio preprocessing
    - Convert audio to Whisper-compatible format (16kHz, mono)
    - Trim silence to reduce processing time
    - Handle long recordings (split if > 10min)
  - [ ] Subtask 7.3: Implement device-specific optimizations
    - Use hardware acceleration (GPU) if available
    - Adjust model size based on device capabilities
    - Warn users on low-end devices (expected longer times)

- [ ] **Task 8: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 8.1: Unit tests for TranscriptionService
    - Test transcribe() with sample audio files
    - Test error handling (corrupted audio, model errors)
    - Test performance monitoring
  - [ ] Subtask 8.2: Unit tests for TranscriptionQueueService
    - Test queue FIFO order
    - Test queue persistence across restarts
    - Test background processing
  - [ ] Subtask 8.3: Integration tests for transcription flow
    - Test end-to-end: audio capture → transcription → result
    - Test offline transcription (no network)
    - Test model download flow
    - Test retry logic (manual and automatic)
  - [ ] Subtask 8.4: Performance tests
    - Verify < 2x audio duration for various lengths (30s, 2min, 5min)
    - Test memory usage during transcription
    - Test device responsiveness (UI remains smooth)
  - [ ] Subtask 8.5: Edge case tests
    - Test very short audio (< 1s)
    - Test very long audio (> 10min)
    - Test rapid successive transcriptions
    - Test app backgrounding during transcription

## Dev Notes

### Architecture Context

**Bounded Context:** Normalization Context (Supporting Domain)
**Domain Services:** TranscriptionService (stateless)
**Supporting Services:** TranscriptionQueueService

**Key Architectural Decision:**
- Transcription is a **transformation** from audio → text
- The Normalization Context is generic (handles audio, image OCR, web scraping)
- MVP focuses on audio transcription only

### Technical Stack

**Whisper Integration:**
- **Library:** `whisper.rn` or `react-native-whisper` (research best option)
- **Model:** Whisper "tiny" or "base" (trade-off speed vs. accuracy)
- **Model size:** ~40-150MB (tiny: 40MB, base: 150MB)
- **Download:** On first launch or lazy (user-triggered)

**React Native Requirements:**
- **Expo custom dev client** (Whisper requires native modules)
- **Increased heap size** (Android: largeHeap=true, iOS: increase memory limit)
- **Background processing** (react-native-background-actions or expo-task-manager)

**Already installed:**
- WatermelonDB (Capture entity)
- expo-file-system (audio file access)

### Performance Requirements

**NFR2:** Transcription < 2x audio duration
- Example: 60s audio → transcription completes in < 120s
- Challenge: Depends on device performance (newer devices faster)
- Strategy: Use smaller Whisper model ("tiny" or "base")

**Device Responsiveness:**
- Transcription runs in background (don't block UI)
- Use React Native's async processing
- Monitor memory usage (Whisper loads model into RAM)

### Confidentialité & Offline-First

**Critical Constraint: 100% Local Processing**
- Whisper model runs entirely on-device
- NO network calls during transcription
- Audio data NEVER leaves the device
- Complies with NFR11 (confidentialité audio)

**Offline Availability:**
- Transcription works identically offline/online
- Model downloaded once, cached locally
- No dependency on external APIs

### Whisper Model Selection

**Trade-offs:**
| Model | Size | Speed | Accuracy | Recommendation |
|-------|------|-------|----------|----------------|
| tiny  | 40MB | Fast (< 2x) | Lower | MVP default |
| base  | 150MB | Medium (≈ 2x) | Medium | Upgrade option |
| small | 500MB | Slow (> 2x) | High | Not for MVP |

**Decision: Use "tiny" for MVP**
- Meets NFR2 performance target
- Smaller download size
- Acceptable accuracy for MVP
- Can upgrade to "base" post-MVP

### Memory Management

**Whisper Model Loading:**
- Model loaded once into memory (cached)
- Released when app backgrounds (iOS memory pressure)
- Reloaded on next transcription if needed

**Long Audio Handling:**
- Split audio files > 10min into chunks
- Transcribe chunks separately
- Concatenate results

### Queue Architecture

**TranscriptionQueueService:**
```typescript
class TranscriptionQueueService {
  queue: Capture[]  // FIFO order
  isProcessing: boolean

  enqueue(capture: Capture): void
  processNext(): Promise<void>
  pause(): void  // App backgrounding
  resume(): void  // App foregrounding
}
```

**Processing Flow:**
```
Capture Recorded → Enqueue → Process (Background) → Update Capture → Notify User
```

### Error Handling Strategy

**Failure Scenarios:**
1. **Model loading error:** Retry or prompt model re-download
2. **Audio format error:** Show user-friendly error, preserve audio
3. **Timeout (> 2x):** Cancel, mark failed, allow retry
4. **Memory error:** Reduce model size or chunk audio

**User-Facing Errors:**
- "Transcription failed. Tap to retry."
- "Audio file corrupted. Please re-record."
- "Device storage full. Free up space and retry."

### Reuse from Stories 2.1-2.4

**Already implemented:**
- Capture WatermelonDB model (state machine)
- Audio file storage (expo-file-system)
- Background queue pattern (SyncQueueService from 2.4)
- Offline-first architecture

**Extend:**
- Add TranscriptionService (new)
- Add TranscriptionQueueService (similar to SyncQueueService)
- Extend Capture states ("processing", "failed")

### File Structure

```
apps/mobile/
├── src/
│   ├── contexts/
│   │   ├── Capture/
│   │   │   └── domain/
│   │   │       └── Capture.model.ts  # EXTEND with transcription states
│   │   └── Normalization/  # NEW Bounded Context
│   │       ├── services/
│   │       │   ├── TranscriptionService.ts  # NEW
│   │       │   ├── TranscriptionQueueService.ts  # NEW
│   │       │   └── WhisperModelService.ts  # NEW (model download/management)
│   │       └── config/
│   │           └── whisper.config.ts  # NEW (model paths, params)
```

### Testing Standards

- **Unit tests:** TranscriptionService, queue, model loading
- **Integration tests:** Full audio → transcription flow
- **Performance tests:** Verify < 2x duration (NFR2)
- **Memory tests:** Monitor memory usage during transcription
- **Offline tests:** Verify no network calls
- **Edge cases:** Short/long audio, rapid transcriptions, app backgrounding

### Dependencies

**New dependencies:**
- `whisper.rn` or `react-native-whisper` (research best option)
- `react-native-background-actions` or `expo-task-manager` (background processing)

**Already installed:**
- `@nozbe/watermelondb`
- `expo-file-system`
- `expo-notifications` (for completion notifications)

### Security & Privacy

**NFR11 Compliance (Confidentialité Audio):**
- Whisper runs 100% locally
- No cloud transcription APIs (Google, AWS, OpenAI)
- Audio files never transmitted off-device
- Transcriptions stored locally (encrypted at rest via NFR12)

### Previous Story Intelligence (Stories 2.1-2.4)

**Learnings:**
- Background queue pattern works well (SyncQueueService)
- Offline-first architecture solid
- Crash recovery mechanism reliable
- State machine pattern for Capture entity

**Code patterns to reuse:**
- Queue service architecture
- Background processing pattern
- Capture state transitions
- Error handling with retry logic

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.5]
- [Source: _bmad-output/planning-artifacts/architecture.md#Normalization Context]
- [Source: _bmad-output/planning-artifacts/architecture.md#Whisper Integration]
- [Source: _bmad-output/implementation-artifacts/2-4-stockage-offline-des-captures.md#Queue Services]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->
