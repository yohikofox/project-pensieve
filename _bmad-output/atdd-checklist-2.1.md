# ATDD Checklist - Story 2.1: Capture Audio 1-Tap

**Status**: 🔴 RED PHASE - All tests failing (expected)

**Story**: 2.1 - Capture Audio 1-Tap
**Epic**: 2 - Capture & Transcription de Pensées
**Test Framework**: Detox E2E (React Native)
**Generated**: 2026-01-20
**Author**: TEA (Test Engineering Architect - Murat)

---

## Summary

**✅ ATDD Setup Complete**

- **Framework**: Detox E2E configured for React Native + Expo
- **Test File**: `pensieve/mobile/e2e/story-2-1-capture-audio.e2e.ts`
- **Total Tests**: 15 E2E tests
- **Coverage**: 100% of 5 acceptance criteria
- **Status**: 🔴 All tests in RED phase (failing as expected)

---

## Test Coverage Breakdown

### AC1: Start Recording with < 500ms Latency (5 tests)

1. ✅ `should start audio recording within 500ms after tap`
   - **Validates**: NFR1 (< 500ms latency)
   - **Method**: Performance measurement with `measurePerformance()` helper
   - **Assertion**: `expect(latency).toBeLessThan(500)`

2. ✅ `should display pulsing red indicator during recording`
   - **Validates**: Visual feedback requirement
   - **TestID**: `recording-indicator`

3. ✅ `should trigger haptic feedback on iOS/Android`
   - **Validates**: Haptic feedback (indirect via recording state)
   - **Note**: Haptics not directly testable in Detox

4. ✅ `should create Capture entity with status "recording"`
   - **Validates**: OP-SQLite entity creation
   - **TestID**: `recording-indicator`, `recording-timer`

5. ✅ `should stream audio data to local storage`
   - **Validates**: File streaming during recording
   - **Validated in AC2**: File existence confirmed after save

### AC2: Stop and Save Recording (4 tests)

6. ✅ `should stop recording immediately when stop button is tapped`
   - **TestID**: `stop-button`, `recording-indicator` (disappears)

7. ✅ `should save audio file to device storage`
   - **TestID**: `capture-feed`, `capture-item-0`

8. ✅ `should update Capture entity with status "captured" and file path`
   - **TestID**: `capture-item-0-metadata`

9. ✅ `should store audio file metadata (duration, size, timestamp)`
   - **TestID**: `capture-item-0-duration`
   - **Assertion**: Duration text (e.g., "00:02")

### AC3: Offline Functionality (2 tests)

10. ✅ `should work identically without network connectivity`
    - **Validates**: NFR7 (100% offline availability)
    - **Method**: `goOffline()` helper disables network
    - **Assertion**: No errors, capture saved locally

11. ✅ `should mark Capture entity for future sync`
    - **TestID**: `capture-item-0-sync-pending`
    - **Validates**: syncStatus = 'pending' in OP-SQLite

### AC4: Crash Recovery (2 tests)

12. ✅ `should recover partial recording after app crash`
    - **Validates**: NFR8 (crash recovery, zero data loss)
    - **Method**: `terminateApp()` → `launchApp()`
    - **Timeout**: 10s (recovery takes time)

13. ✅ `should notify user about recovered capture`
    - **TestID**: `recovery-notification`

### AC5: Microphone Permission Handling (2 tests)

14. ✅ `should prompt for microphone access when permission not granted`
    - **Method**: Launch with `permissions: { microphone: 'NO' }`
    - **TestID**: `permission-required-message`

15. ✅ `should only start recording after permission is granted`
    - **Method**: Launch with `permissions: { microphone: 'YES' }`
    - **TestID**: `recording-indicator`

---

## Required data-testid Attributes

**⚠️ CRITICAL**: Implementation MUST add these testIDs to React Native components.

### Main Screen Components

```tsx
// Main screen container
<View testID="main-screen">
  {/* Record button */}
  <TouchableOpacity testID="record-button" onPress={startRecording}>
    <Text>Record</Text>
  </TouchableOpacity>

  {/* Stop button (visible during recording) */}
  <TouchableOpacity testID="stop-button" onPress={stopRecording}>
    <Text>Stop</Text>
  </TouchableOpacity>
</View>
```

### Recording State Components

```tsx
// Recording indicator (pulsing red dot)
<View testID="recording-indicator" style={styles.pulsingRed}>
  <Text>● REC</Text>
</View>

// Recording timer
<Text testID="recording-timer">00:05</Text>
```

### Capture Feed Components

```tsx
// Feed container
<FlatList testID="capture-feed" data={captures} renderItem={...} />

// Individual capture items (use index)
<View testID={`capture-item-${index}`}>
  {/* Metadata container */}
  <View testID={`capture-item-${index}-metadata`}>
    <Text testID={`capture-item-${index}-duration`}>00:02</Text>
  </View>

  {/* Sync pending badge */}
  {capture.syncStatus === 'pending' && (
    <View testID={`capture-item-${index}-sync-pending`}>
      <Text>⏳ Pending sync</Text>
    </View>
  )}
</View>
```

### Error & Notification Components

```tsx
// General error message
{error && <Text testID="error-message">{error}</Text>}

// Network error (should NOT appear in offline tests)
{networkError && <Text testID="network-error">{networkError}</Text>}

// Permission required message
{!hasMicPermission && (
  <Text testID="permission-required-message">
    Microphone access required
  </Text>
)}

// Crash recovery notification
{recoveredCapture && (
  <View testID="recovery-notification">
    <Text>Recording recovered</Text>
  </View>
)}
```

---

## Implementation Checklist

### Prerequisites

- [ ] Run `npm install` in `pensieve/mobile/` to install Detox
- [ ] Run `npm run prebuild:clean` to generate iOS/Android folders
- [ ] Run `npm run test:e2e:build:ios` to build test app
- [ ] Verify tests fail: `npm run test:e2e` (Expected: 15 failures)

### Task 1: Setup Capture Context Mobile Infrastructure

#### Subtask 1.1: Create Capture OP-SQLite Model

- [ ] Define `Capture.schema.ts` with fields:
  - `id`, `type`, `state`, `rawContent`, `normalizedText`
  - `capturedAt`, `location`, `tags`, `syncStatus`
- [ ] Implement `Capture.model.ts` with OP-SQLite decorators
- [ ] Create migration script for Capture table
- [ ] **TestIDs added**: N/A (database model)
- [ ] **Tests affected**: AC1 test #4, AC2 test #8

#### Subtask 1.2: Configure expo-av Permissions

- [ ] Add microphone permission to `app.json`
- [ ] Implement permission request flow in UI
- [ ] Handle permission denied gracefully
- [ ] **TestIDs added**: `permission-required-message`
- [ ] **Tests affected**: AC5 tests #14, #15
- [ ] **Run test**: `detox test e2e/story-2-1-capture-audio.e2e.ts -t "AC5"`

#### Subtask 1.3: Implement RecordingService

- [ ] Create `RecordingService.ts` with expo-av
- [ ] Implement `startRecording()` method
- [ ] Implement `stopRecording()` method
- [ ] Configure audio format (m4a), quality, channels
- [ ] Save files to `FileSystem.documentDirectory`
- [ ] File naming: `capture_{userId}_{timestamp}_{uuid}.m4a`
- [ ] **TestIDs added**: N/A (service layer)
- [ ] **Tests affected**: All AC1 and AC2 tests

#### Subtask 1.4: Create RecordButton Component

- [ ] Implement `RecordButton.tsx` component
- [ ] Add pulsing red indicator animation (60fps)
- [ ] Integrate expo-haptics for tap feedback
- [ ] Add recording timer display
- [ ] **TestIDs added**:
  - `record-button`
  - `stop-button`
  - `recording-indicator`
  - `recording-timer`
- [ ] **Tests affected**: AC1 tests #1-5, AC2 test #6
- [ ] **Run tests**: `npm run test:e2e` (AC1 + AC2 should start passing)

### Task 2: Implement Offline-First Capture Persistence

#### Subtask 2.1: Auto-save During Recording

- [ ] Stream audio to temporary file during recording
- [ ] Implement atomic file operations
- [ ] Handle low storage scenarios
- [ ] **TestIDs added**: N/A (background service)
- [ ] **Tests affected**: AC2 test #7

#### Subtask 2.2: Crash Recovery Mechanism

- [ ] Detect incomplete recordings on app launch
- [ ] Implement `CrashRecoveryService.ts`
- [ ] Attempt recovery of partial files
- [ ] Store recovery metadata in OP-SQLite
- [ ] Show notification to user
- [ ] **TestIDs added**: `recovery-notification`
- [ ] **Tests affected**: AC4 tests #12, #13
- [ ] **Run tests**: `detox test e2e/story-2-1-capture-audio.e2e.ts -t "AC4"`

#### Subtask 2.3: Mark Captures for Sync

- [ ] Add `syncStatus` field to Capture model
- [ ] Set `syncStatus='pending'` for offline captures
- [ ] Implement offline queue
- [ ] Display sync badge in UI
- [ ] **TestIDs added**: `capture-item-{index}-sync-pending`
- [ ] **Tests affected**: AC3 test #11
- [ ] **Run tests**: `detox test e2e/story-2-1-capture-audio.e2e.ts -t "AC3"`

### Task 3: Implement Audio File Storage Management

#### Subtask 3.1: File Naming Convention

- [ ] Implement naming: `capture_{userId}_{timestamp}_{uuid}.m4a`
- [ ] Store in secure directory
- [ ] **TestIDs added**: N/A (file system)
- [ ] **Tests affected**: AC2 test #7

#### Subtask 3.2: Store Metadata

- [ ] Record duration, file size, timestamp
- [ ] Store file path in `Capture.rawContent`
- [ ] Display metadata in feed
- [ ] **TestIDs added**:
  - `capture-item-{index}-metadata`
  - `capture-item-{index}-duration`
- [ ] **Tests affected**: AC2 tests #8, #9
- [ ] **Run tests**: `detox test e2e/story-2-1-capture-audio.e2e.ts -t "AC2"`

### Task 4: Create Capture Feed UI

- [ ] Implement `CaptureFeed.tsx` component
- [ ] Display captures in chronological order
- [ ] Show metadata (duration, timestamp)
- [ ] Show sync status badges
- [ ] **TestIDs added**:
  - `capture-feed`
  - `capture-item-{index}`
  - `capture-item-{index}-metadata`
  - `capture-item-{index}-duration`
  - `capture-item-{index}-sync-pending`
- [ ] **Tests affected**: AC2 tests #7-9, AC3 tests #10-11
- [ ] **Run tests**: `npm run test:e2e`

---

## Running Tests

### Run all E2E tests

```bash
cd pensieve/mobile
npm run test:e2e
```

**Expected**: 15 failures initially (RED phase)

### Run specific AC tests

```bash
# AC1 only (5 tests)
detox test e2e/story-2-1-capture-audio.e2e.ts -t "AC1"

# AC2 only (4 tests)
detox test e2e/story-2-1-capture-audio.e2e.ts -t "AC2"

# Performance test (NFR1 < 500ms)
detox test e2e/story-2-1-capture-audio.e2e.ts -t "should start audio recording within 500ms"
```

### Debug specific test

```bash
detox test e2e/story-2-1-capture-audio.e2e.ts -t "should display pulsing red indicator" --loglevel verbose
```

---

## Red-Green-Refactor Workflow

### 🔴 RED Phase (Complete)

✅ **15 failing E2E tests created**
✅ **Helpers and infrastructure setup**
✅ **Test documentation complete**

**Status**: Ready for DEV implementation

### 🟢 GREEN Phase (DEV Team - In Progress)

**Workflow**:
1. Pick one failing test (start with AC1 test #1)
2. Implement minimal code to make it pass
3. Add required `data-testid` attributes
4. Run test: `detox test ... -t "test name"`
5. If test passes ✅, move to next test
6. If test fails ❌, fix implementation
7. Repeat until all 15 tests pass

**Tip**: Implement in order: AC1 → AC2 → AC5 → AC3 → AC4
(AC4 crash recovery is most complex, do last)

### 🔵 REFACTOR Phase (DEV Team - After GREEN)

Once all tests pass:
1. Refactor code (extract helpers, optimize)
2. Ensure tests still pass
3. Improve performance if needed
4. Tests provide safety net

---

## Performance Validation

**NFR1**: < 500ms latency from tap to recording start

Test #1 validates this with performance measurement:

```typescript
const latency = await measurePerformance('tap-to-record', async () => {
  await tapElement('record-button');
  await waitForElement('recording-indicator', 1000);
});

expect(latency).toBeLessThan(500); // MUST pass
```

If test fails (latency > 500ms), optimize:
- Reduce RecordingService initialization time
- Optimize component re-renders
- Profile with React Native Performance Monitor

---

## Offline Testing Validation

**NFR7**: 100% offline availability

Tests #10-11 validate offline-first:

```typescript
await goOffline(); // Network disabled
await tapElement('record-button'); // Should work
await expectVisible('capture-item-0'); // Saved locally
await expectVisible('capture-item-0-sync-pending'); // Marked for sync
```

---

## Zero Data Loss Validation

**NFR6/NFR8**: Zero data loss, crash recovery

Tests #12-13 validate crash recovery:

```typescript
await tapElement('record-button');
await device.sleep(2000); // Record 2s
await terminateApp(); // CRASH
await launchApp(); // Relaunch
await expectVisible('recovery-notification'); // User notified
// Partial recording recovered
```

---

## Knowledge Base Patterns Applied

- ✅ **Given-When-Then** structure (`test-quality.md`)
- ✅ **Deterministic waits** (`timing-debugging.md`)
- ✅ **data-testid selectors** (`selector-resilience.md`)
- ✅ **One assertion per test** (`test-quality.md`)
- ✅ **Performance measurement** helper for NFR1
- ✅ **Offline testing** with network simulation

---

## Next Steps

1. ✅ ATDD setup complete
2. ✅ Tests in RED phase
3. ⏳ **DEV Team**: Implement Story 2.1 following checklist
4. ⏳ **DEV Team**: Add all required `data-testid` attributes
5. ⏳ **DEV Team**: Run tests frequently, aim for GREEN
6. ⏳ **DEV Team**: Refactor after GREEN with confidence

---

## Support

**Questions?** Refer to:
- `pensieve/mobile/e2e/README.md` - Detox setup guide
- Story 2.1 file - Dev Notes section
- TEA knowledge base - `_bmad/bmm/testarch/knowledge/`

**Test failing unexpectedly?** Check:
1. Missing `data-testid` attribute
2. Component not rendering
3. Detox build out of date (rebuild: `npm run test:e2e:build:ios`)
