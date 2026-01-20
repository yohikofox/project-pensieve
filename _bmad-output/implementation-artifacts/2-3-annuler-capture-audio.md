# Story 2.3: Annuler Capture Audio

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **to cancel an audio recording in progress**,
so that **I can discard unwanted captures without cluttering my feed**.

## Acceptance Criteria

### AC1: Cancel Recording with Immediate Stop
**Given** I am currently recording audio
**When** I tap the cancel button
**Then** the recording stops immediately
**And** the audio file is deleted from device storage
**And** the Capture entity is removed from WatermelonDB
**And** I am returned to the main screen ready for a new capture

### AC2: Cancel Gesture with Confirmation
**Given** I am recording audio
**When** I swipe down or perform the cancel gesture
**Then** a confirmation prompt appears "Discard this recording?"
**And** if I confirm, the recording is cancelled (same behavior as cancel button)
**And** if I decline, the recording continues

### AC3: Haptic Feedback on Cancellation
**Given** I cancel a recording
**When** the cancellation is processed
**Then** haptic feedback indicates the cancellation (UX requirement)
**And** a brief animation shows the capture being discarded

### AC4: Accidental Cancel Protection
**Given** I accidentally tap cancel during recording
**When** the confirmation prompt appears
**Then** I can choose to continue recording
**And** the recording resumes without data loss

### AC5: Offline Cancellation Support
**Given** the app is offline
**When** I cancel a recording
**Then** the cancellation works identically to online mode (FR4 compliance)
**And** no orphaned files remain in storage

## Tasks / Subtasks

- [ ] **Task 1: Add Cancel Button to Recording UI** (AC: 1, 3)
  - [ ] Subtask 1.1: Design cancel button component
    - Place cancel button prominently during recording state
    - Use clear "X" or "Cancel" icon with red/warning color
    - Ensure accessible tap target (44x44 min)
    - Show button only when recording state is active
  - [ ] Subtask 1.2: Implement cancel button action
    - Trigger cancellation flow on tap
    - Stop recording via RecordingService
    - Delete partial audio file from storage
    - Remove Capture entity from WatermelonDB
  - [ ] Subtask 1.3: Add haptic feedback
    - Use expo-haptics with "warning" or "error" impact
    - Trigger immediately on cancel confirmation
    - Provide distinct feel from save haptic

- [ ] **Task 2: Implement Swipe-Down Cancel Gesture** (AC: 2, 4)
  - [ ] Subtask 2.1: Add gesture recognizer
    - Use React Native PanResponder or Gesture Handler
    - Detect vertical swipe-down during recording
    - Set threshold distance (e.g., 100px swipe)
  - [ ] Subtask 2.2: Show confirmation dialog
    - Display native Alert: "Discard this recording?"
    - Options: "Discard" | "Keep Recording"
    - Pause recording timer during dialog (optional UX)
  - [ ] Subtask 2.3: Handle user response
    - If "Discard": Execute same cancellation flow as button
    - If "Keep Recording": Dismiss dialog, continue recording
    - Ensure no audio data loss during pause/resume

- [ ] **Task 3: Implement Cancellation Logic in RecordingService** (AC: 1, 5)
  - [ ] Subtask 3.1: Add cancelRecording() method
    - Stop expo-av recording immediately
    - Return partial file path for deletion
    - Update Capture entity state to "cancelled" (transient)
  - [ ] Subtask 3.2: Delete partial audio file
    - Use expo-file-system to delete file atomically
    - Handle file not found errors gracefully
    - Verify file deletion success
  - [ ] Subtask 3.3: Remove Capture entity from WatermelonDB
    - Delete transient Capture record
    - Ensure no orphaned records in DB
    - Handle concurrent operations safely
  - [ ] Subtask 3.4: Clean up storage completely
    - Verify no temporary files remain
    - Check for orphaned metadata
    - Works identically offline (AC5)

- [ ] **Task 4: Implement Discard Animation** (AC: 3)
  - [ ] Subtask 4.1: Design discard animation
    - Fade out + shrink animation for recording indicator
    - Duration: 200-300ms (feels responsive)
    - Follow Liquid Glass design principles (60fps)
  - [ ] Subtask 4.2: Return to ready state
    - Transition smoothly to main screen
    - Reset recording button to default state
    - Prepare UI for next capture immediately

- [ ] **Task 5: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 5.1: Unit tests for cancelRecording()
    - Test recording stop and file deletion
    - Test Capture entity removal
    - Test cleanup of storage/metadata
    - Test offline cancellation
  - [ ] Subtask 5.2: Component tests for cancel UI
    - Test cancel button visibility during recording
    - Test haptic feedback trigger
    - Test discard animation
  - [ ] Subtask 5.3: Integration tests for cancellation flows
    - Test button cancel flow (immediate)
    - Test swipe-down gesture cancel flow
    - Test confirmation dialog ("Discard" vs "Keep Recording")
    - Test no data loss when choosing "Keep Recording"
  - [ ] Subtask 5.4: Edge case tests
    - Test cancellation during very short recordings (< 1s)
    - Test cancellation during very long recordings
    - Test rapid successive cancel attempts
    - Test offline cancellation leaves no orphans

## Dev Notes

### Architecture Context

**Bounded Context:** Capture Context (Supporting Domain)
**Aggregate:** `Capture` (cancelled state is transient)
**Domain Events:** `CaptureAborted` (optional - could be logged)

**Cancellation Flow:**
```
Recording State → Cancel Trigger → Stop Recording → Delete File → Remove Entity → Ready State
```

### Technical Stack

**Reuse from Story 2.1:**
- RecordingService (extend with cancelRecording method)
- Capture model (add transient "cancelled" state)
- expo-haptics (different impact for cancel)
- expo-file-system (file deletion)

**Additional:**
- React Native Gesture Handler or PanResponder (swipe gesture)
- Alert API (confirmation dialog)
- Animation API (discard animation)

### Performance Requirements

**NFR6:** 0 capture perdue, jamais
- Cancellation must be intentional (confirmation required)
- No accidental data loss

**NFR7:** Disponibilité capture offline = 100%
- Cancellation works identically offline

### UX Requirements (Liquid Glass Design System)

- **Cancel button:** Clear and accessible, red/warning color
- **Swipe gesture:** Natural vertical swipe-down to dismiss
- **Confirmation dialog:** Native modal, clear options
- **Haptic feedback:** "Warning" impact (stronger than save)
- **Discard animation:** Smooth fade + shrink, 60fps, 200-300ms
- **No data loss:** "Keep Recording" option preserves all data

### Reuse from Story 2.1

**Already implemented:**
- RecordingService architecture
- expo-av recording integration
- expo-haptics setup
- Capture entity in WatermelonDB
- File storage management

**Extend:**
- Add cancelRecording() to RecordingService
- Add cancel button to RecordButton component
- Add swipe gesture recognizer

### File Structure

```
apps/mobile/
├── src/
│   ├── contexts/
│   │   └── Capture/
│   │       ├── services/
│   │       │   └── RecordingService.ts  # EXTEND with cancelRecording()
│   │       └── ui/
│   │           └── RecordButton.tsx  # ADD cancel button & swipe gesture
```

### Testing Standards

- Unit tests for cancelRecording() method
- Component tests for cancel button/gesture
- Integration tests for full cancel flows
- Edge case tests (short/long recordings, rapid cancel)
- Offline mode tests
- No orphaned files/entities verification

### Dependencies

**Already installed (from Story 2.1):**
- `expo-av`
- `expo-haptics`
- `expo-file-system`
- `@nozbe/watermelondb`

**Consider adding:**
- `react-native-gesture-handler` (if not already installed via Expo)

### Previous Story Intelligence (Story 2.1)

**Learnings:**
- RecordingService pattern established
- File management with expo-file-system
- Capture entity lifecycle (recording → captured)
- Haptic feedback integration successful

**Code patterns to reuse:**
- Service method architecture
- File deletion with error handling
- WatermelonDB entity removal
- Haptic feedback patterns

**New patterns:**
- Confirmation dialog for destructive actions
- Gesture recognition for UX enhancement
- Transient state handling (cancelled)

### Security Considerations

**NFR6 Compliance (Zero data loss):**
- Cancellation is intentional (requires confirmation)
- "Keep Recording" option prevents accidental loss
- No silent failures - all errors logged

**Storage cleanup:**
- Atomic file deletion
- No orphaned temporary files
- Verify deletion success

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.3]
- [Source: _bmad-output/planning-artifacts/architecture.md#Capture Context]
- [Source: _bmad-output/implementation-artifacts/2-1-capture-audio-1-tap.md#RecordingService]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->
