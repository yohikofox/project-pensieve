# ATDD Checklist: Story 2.1 - Capture Audio 1-Tap

**Date:** 2026-01-21
**Status:** 🔴 RED Phase (Tests Failing)
**Story File:** `_bmad-output/implementation-artifacts/2-1-capture-audio-1-tap.md`

---

## Story Summary

**As a** user
**I want** to record an audio thought with a single tap from the main screen
**So that** I can quickly capture my ideas without friction, even without network connectivity

---

## Acceptance Criteria Breakdown

### ✅ AC1: Start Recording with < 500ms Latency

**Requirements:**
- Audio recording starts within 500ms (NFR1 compliance)
- Visual feedback displayed (pulsing red indicator)
- Haptic feedback triggered on iOS/Android
- Capture entity created in WatermelonDB with status "recording"
- Audio data streamed to local storage

**Test Coverage:**
- ✅ E2E: `audio-1-tap.e2e.ts` - "should start audio recording within 500ms of tap"
- ✅ E2E: `audio-1-tap.e2e.ts` - "should display visual feedback (pulsing red indicator)"
- ✅ E2E: `audio-1-tap.e2e.ts` - "should trigger haptic feedback on iOS/Android"
- ✅ Integration: `recording-service.test.ts` - "should start recording and return within 500ms"
- ✅ Integration: `capture-model.test.ts` - "should create Capture with correct schema"

### ✅ AC2: Stop and Save Recording

**Requirements:**
- Recording stops immediately
- Audio file saved to device storage
- Capture entity updated with status "captured" and file path
- Audio file metadata (duration, size, timestamp) stored

**Test Coverage:**
- ✅ E2E: `audio-1-tap.e2e.ts` - "should stop recording immediately when stop button tapped"
- ✅ E2E: `audio-1-tap.e2e.ts` - "should save audio file to device storage"
- ✅ Integration: `recording-service.test.ts` - "should stop recording immediately"
- ✅ Integration: `capture-model.test.ts` - "should transition state from 'recording' to 'captured'"

### ✅ AC3: Offline Functionality

**Requirements:**
- Works identically to online mode (FR4, NFR7 compliance)
- Capture entity marked for future sync
- No error shown to user

**Test Coverage:**
- ✅ E2E: `audio-1-tap.e2e.ts` - "should work identically in offline mode"
- ✅ Integration: `recording-service.test.ts` - "should work without network connectivity"
- ✅ Integration: `capture-model.test.ts` - "should mark new captures as 'pending' sync"

### ✅ AC4: Crash Recovery

**Requirements:**
- Partial recording recovered if possible (NFR8 compliance)
- User notified about recovered capture

**Test Coverage:**
- ✅ E2E: `audio-1-tap.e2e.ts` - "should recover partial recording after crash"
- ✅ Integration: `recording-service.test.ts` - "should detect incomplete recordings on initialization"
- ✅ Integration: `recording-service.test.ts` - "should recover partial audio file if valid"

### ✅ AC5: Microphone Permission Handling

**Requirements:**
- User prompted to grant microphone access
- Recording only starts after permission granted

**Test Coverage:**
- ✅ E2E: `audio-1-tap.e2e.ts` - "should prompt for microphone permission if not granted"
- ✅ Integration: `recording-service.test.ts` - "should check microphone permission before recording"

---

## Test Files Created

### E2E Tests (Detox)
```
pensieve/mobile/e2e/capture/audio-1-tap.e2e.ts
```
- **Test Count**: 11 tests
- **Run Command**: `detox test --configuration ios.sim.debug`
- **Status**: 🔴 All failing (waiting for implementation)

### Integration Tests (Jest)
```
pensieve/mobile/tests/acceptance/capture/recording-service.test.ts
pensieve/mobile/tests/acceptance/capture/capture-model.test.ts
```
- **Test Count**: 22 tests
- **Run Command**: `npm run test:acceptance`
- **Status**: 🔴 All failing (waiting for implementation)

### Supporting Infrastructure

**Data Factory:**
```
pensieve/mobile/tests/support/factories/capture.factory.ts
```
- Factory methods: `create`, `createAudioCapture`, `createRecordingCapture`, `build`, `buildMany`
- Uses faker for random data
- Auto-cleanup methods provided

---

## Required data-testid Attributes

### Main Screen
- `main-screen` - Main screen container
- `record-button` - Record button (tap to start)
- `stop-button` - Stop button (tap to end recording)

### Recording Indicators
- `recording-indicator` - Visual pulsing red indicator
- `recording-timer` - Recording duration timer (displays 00:00 format)

### Capture Status
- `capture-saved-indicator` - Success indicator after save
- `sync-pending-indicator` - Shows when capture awaiting sync
- `recovery-notification` - Crash recovery notification
- `permission-rationale` - Microphone permission explanation

### Capture Details View
- `view-capture-details` - Button to view capture metadata
- `capture-duration` - Audio duration display
- `capture-size` - File size display
- `capture-timestamp` - Capture timestamp display

---

## Implementation Checklist

### 🔴 RED Phase Complete

- [x] All tests written (33 tests total)
- [x] Tests failing for right reason (missing implementation)
- [x] Data factories created
- [x] ATDD checklist generated

### 🟢 GREEN Phase (Implementation Tasks)

#### Task 1: Setup Capture Context Mobile Infrastructure

- [ ] **Subtask 1.1**: Create Capture aggregate model in WatermelonDB
  - [ ] Define schema with fields: id, type, state, rawContent, normalizedText, capturedAt, location, tags, syncStatus
  - [ ] Implement WatermelonDB model with decorators
  - [ ] Add migration script for Capture table
  - [ ] Run tests: `npm run test:acceptance -- capture-model.test.ts`
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 1.2**: Configure expo-av audio recording permissions
  - [ ] Add microphone permission to app.json
  - [ ] Implement permission request flow
  - [ ] Handle permission denied state
  - [ ] Run tests: `npm run test:acceptance -- recording-service.test.ts` (permission tests)
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 1.3**: Implement RecordingService with expo-av
  - [ ] Create RecordingService class
  - [ ] Configure recording options (format, quality, channels)
  - [ ] Implement startRecording() method
  - [ ] Implement stopRecording() method
  - [ ] Save audio files to secure device storage
  - [ ] Run tests: `npm run test:acceptance -- recording-service.test.ts`
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 1.4**: Create React component for record button UI
  - [ ] Create RecordButton.tsx component
  - [ ] Implement pulsing red indicator animation
  - [ ] Add haptic feedback using expo-haptics
  - [ ] Add data-testid attributes (record-button, recording-indicator, etc.)
  - [ ] Ensure < 500ms tap-to-record latency (NFR1)
  - [ ] Display recording duration timer
  - [ ] Run tests: `detox test -- audio-1-tap.e2e.ts` (AC1 tests)
  - [ ] ✅ Tests pass (green)

#### Task 2: Implement Offline-First Capture Persistence

- [ ] **Subtask 2.1**: Implement auto-save during recording
  - [ ] Stream audio data to temporary file during recording
  - [ ] Use atomic file operations to prevent corruption
  - [ ] Handle low storage scenarios gracefully
  - [ ] Run tests: `npm run test:acceptance -- recording-service.test.ts`
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 2.2**: Implement crash recovery mechanism
  - [ ] Create CrashRecoveryService class
  - [ ] Detect incomplete recordings on app launch
  - [ ] Attempt recovery of partial audio files
  - [ ] Store recovery metadata in WatermelonDB
  - [ ] Notify user of recovered captures
  - [ ] Run tests: `detox test -- audio-1-tap.e2e.ts` (AC4 tests)
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 2.3**: Mark Capture entities for sync
  - [ ] Add syncStatus field handling
  - [ ] Implement offline queue for pending captures
  - [ ] Ensure WatermelonDB sync protocol compatibility
  - [ ] Run tests: `npm run test:acceptance -- capture-model.test.ts` (sync tests)
  - [ ] ✅ Tests pass (green)

#### Task 3: Implement Audio File Storage Management

- [ ] **Subtask 3.1**: Define audio file naming convention
  - [ ] Implement generateFilePath() method
  - [ ] Use pattern: `capture_{userId}_{timestamp}_{uuid}.m4a`
  - [ ] Store in secure directory (FileSystem.documentDirectory)
  - [ ] Run tests: `npm run test:acceptance -- recording-service.test.ts`
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 3.2**: Store metadata with Capture entity
  - [ ] Record duration, file size, timestamp
  - [ ] Store file path reference in Capture.rawContent
  - [ ] Track recording quality/format metadata
  - [ ] Run tests: `detox test -- audio-1-tap.e2e.ts` (AC2 metadata tests)
  - [ ] ✅ Tests pass (green)

#### Task 4: Final Verification

- [ ] **Run all E2E tests**
  - [ ] iOS: `detox test --configuration ios.sim.debug`
  - [ ] Android: `detox test --configuration android.emu.debug`
  - [ ] ✅ All tests pass (green)

- [ ] **Run all integration tests**
  - [ ] `npm run test:acceptance`
  - [ ] ✅ All 22 tests pass (green)

- [ ] **Performance benchmarks**
  - [ ] Verify < 500ms latency (NFR1) on physical devices
  - [ ] Test with various audio durations (30s, 2min, 5min)
  - [ ] Verify memory usage during long recordings

### ♻️ REFACTOR Phase (Post-Implementation)

- [ ] Extract common patterns from RecordingService
- [ ] Optimize recording latency (profile and improve)
- [ ] Add error handling edge cases
- [ ] Improve code documentation
- [ ] Ensure all tests still pass after refactoring

---

## Red-Green-Refactor Workflow

### 🔴 RED Phase (Complete)

**Status**: ✅ All tests written and failing

**What we have:**
- 11 E2E tests (Detox)
- 22 Integration tests (Jest)
- Data factories with faker
- Required data-testid attributes documented

**Next Step**: Start GREEN phase implementation

### 🟢 GREEN Phase (DEV Team)

**Approach**: Implement minimal code to make tests pass

1. Pick one failing test from checklist
2. Implement minimal code to make it pass
3. Run test to verify green
4. Move to next test
5. Repeat until all tests pass

**Key Principles:**
- Don't over-engineer
- Focus on making tests pass
- One test at a time
- Commit after each green

### ♻️ REFACTOR Phase (DEV Team)

**When**: After all tests passing (green)

**Activities:**
1. Extract duplications
2. Improve code quality
3. Optimize performance
4. Add documentation
5. Ensure tests still pass

**Safety Net**: Tests provide confidence for refactoring

---

## Running Tests

### E2E Tests (Detox)

```bash
# Build iOS app
detox build --configuration ios.sim.debug

# Run E2E tests on iOS Simulator
detox test --configuration ios.sim.debug

# Run specific test file
detox test --configuration ios.sim.debug e2e/capture/audio-1-tap.e2e.ts

# Run tests in headed mode (see simulator)
detox test --configuration ios.sim.debug --headless=false

# Debug specific test
detox test --configuration ios.sim.debug --debug-synchronization 200
```

### Integration Tests (Jest)

```bash
# Run all acceptance tests
npm run test:acceptance

# Run specific test file
npm run test:acceptance -- capture-model.test.ts

# Run with coverage
npm run test:acceptance -- --coverage

# Watch mode (re-run on file changes)
npm run test:acceptance -- --watch
```

### Performance Benchmarks

```bash
# TODO: Add performance test script
# npm run test:performance
```

---

## Mock Requirements for DEV Team

### Audio Recording Mocks (Development)

**expo-av Recording Mock:**
- Mock `Audio.Recording` for unit tests
- Mock `Audio.requestPermissionsAsync()` for permission tests
- Simulate recording state (isRecording, durationMillis)

**expo-file-system Mock:**
- Mock `FileSystem.documentDirectory` path
- Mock `FileSystem.getInfoAsync()` for file validation
- Mock `FileSystem.deleteAsync()` for cleanup

**expo-haptics Mock:**
- Mock `Haptics.impactAsync()` for unit tests
- Actual haptic feedback only testable on physical devices

### WatermelonDB Test Setup

**In-memory Database:**
- Use SQLite in-memory adapter for tests
- Auto-reset database between tests
- Seed test data using factories

---

## Dependencies & Libraries Required

### Already Installed (Epic 1)
- ✅ `react-native` + `expo`
- ✅ `@nozbe/watermelondb` (~0.27.0)
- ✅ TypeScript tooling
- ✅ Jest testing framework

### Need to Install

```bash
# Audio recording
expo install expo-av

# Haptic feedback
expo install expo-haptics

# File system
expo install expo-file-system

# Test dependencies
npm install --save-dev @faker-js/faker
npm install --save-dev detox
```

---

## Knowledge Base References Applied

- ✅ **data-factories.md**: Factory patterns with faker, overrides, API seeding
- ✅ **network-first.md**: Route interception patterns (not applicable for mobile)
- ✅ **test-quality.md**: Deterministic tests, cleanup, explicit assertions
- ✅ **test-levels-framework.md**: E2E vs Integration test selection
- ✅ **selector-resilience.md**: data-testid selector strategy
- ✅ **timing-debugging.md**: Race condition prevention, deterministic waiting

---

## Next Steps for DEV Team

1. **Review this checklist** and confirm understanding
2. **Run failing tests** to see RED phase:
   ```bash
   npm run test:acceptance
   detox test --configuration ios.sim.debug
   ```
3. **Start GREEN phase** - Pick first task from Implementation Checklist
4. **Implement one test at a time** (RED → GREEN)
5. **Commit after each green** test
6. **Refactor with confidence** after all tests pass
7. **Share progress** in daily standup

---

## Story Completion Criteria

**Definition of Done:**

- [ ] All 33 tests passing (green)
- [ ] NFR1 verified: < 500ms latency on physical devices
- [ ] NFR6 verified: Zero data loss (crash recovery tested)
- [ ] NFR7 verified: 100% offline availability
- [ ] NFR8 verified: Automatic crash recovery
- [ ] Code review completed
- [ ] Story status updated to "done" in sprint-status.yaml

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `_bmad/bmm/testarch/atdd`
**Version**: 4.0 (BMad v6)
**Date**: 2026-01-21
