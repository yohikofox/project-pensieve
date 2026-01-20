# Story 2.2: Capture Texte Rapide

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **to capture a text thought via quick text input**,
so that **I can record ideas that are better suited to typing than speaking**.

## Acceptance Criteria

### AC1: Open Text Input Field Immediately
**Given** I am on the main screen of the app
**When** I tap the text capture button
**Then** a text input field appears immediately
**And** the keyboard opens automatically
**And** the cursor is focused in the text field

### AC2: Save Text Capture with Metadata
**Given** I have typed text in the capture field
**When** I tap the save/submit button
**Then** a Capture entity is created in WatermelonDB with type "text"
**And** the text content is stored with the Capture
**And** a timestamp is recorded
**And** the Capture status is set to "captured"
**And** the text input field is cleared for the next capture

### AC3: Cancel Unsaved Text with Confirmation
**Given** I start typing a text capture
**When** I cancel or navigate away before saving
**Then** I am prompted to confirm discarding unsaved content
**And** the capture is not saved if I confirm discard

### AC4: Offline Text Capture Functionality
**Given** I have no network connectivity
**When** I create and save a text capture
**Then** the capture is saved locally (FR4 compliance)
**And** the Capture entity is marked for future sync
**And** no error is shown to the user

### AC5: Empty Text Validation
**Given** I submit an empty text field
**When** I attempt to save
**Then** I receive a validation message "Please enter some text"
**And** no empty Capture entity is created

### AC6: Haptic Feedback on Save
**Given** I create a text capture
**When** the save is successful
**Then** subtle haptic feedback confirms the save (UX requirement)
**And** a brief animation shows the capture being added to the feed

## Tasks / Subtasks

- [ ] **Task 1: Create Text Capture UI Component** (AC: 1, 5, 6)
  - [ ] Subtask 1.1: Design TextCaptureInput component
    - Create reusable React Native component with TextInput
    - Auto-focus cursor on mount
    - Auto-open keyboard on component display
    - Implement multiline text input with max height
  - [ ] Subtask 1.2: Add text capture button to main screen
    - Place button alongside audio record button
    - Use distinct icon (keyboard/text icon)
    - Ensure accessible tap target size (44x44 min)
  - [ ] Subtask 1.3: Implement validation UI
    - Show inline error for empty submissions
    - Disable save button when text is empty
    - Clear error on user input
  - [ ] Subtask 1.4: Add haptic feedback on save
    - Use expo-haptics for success confirmation
    - Implement subtle save animation (fade/slide)
    - Show brief "Saved" confirmation toast

- [ ] **Task 2: Implement Text Capture Persistence** (AC: 2, 4)
  - [ ] Subtask 2.1: Create TextCaptureService
    - Handle text capture creation logic
    - Validate non-empty content
    - Generate capture metadata (timestamp, type)
  - [ ] Subtask 2.2: Store text in Capture entity
    - Set type='text' in Capture model
    - Store text content in rawContent field
    - Set normalizedText = rawContent (no transformation needed)
    - Mark syncStatus as 'pending' for offline captures
  - [ ] Subtask 2.3: Clear input after successful save
    - Reset TextInput component state
    - Return focus to input for quick successive captures
    - Provide visual confirmation of save

- [ ] **Task 3: Implement Cancel/Discard Flow** (AC: 3)
  - [ ] Subtask 3.1: Detect unsaved content
    - Track text input state (empty vs. has content)
    - Monitor navigation events (back button, tab switch)
  - [ ] Subtask 3.2: Show discard confirmation dialog
    - Display native alert/modal: "Discard unsaved text?"
    - Options: "Discard" | "Keep Editing"
    - Only show if text is non-empty
  - [ ] Subtask 3.3: Handle discard action
    - Clear text input if user confirms
    - Return to previous screen/state
    - Do not create Capture entity

- [ ] **Task 4: Offline Mode Support** (AC: 4)
  - [ ] Subtask 4.1: Test offline text capture
    - Ensure Capture entity is created in WatermelonDB
    - Verify syncStatus set to 'pending'
    - Confirm no network calls attempted
  - [ ] Subtask 4.2: Queue for sync when online
    - Integrate with existing sync queue (from Story 2.1)
    - Prioritize text captures equally with audio

- [ ] **Task 5: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 5.1: Unit tests for TextCaptureService
    - Test text validation (empty, whitespace-only, valid)
    - Test Capture entity creation with correct type
    - Test metadata generation
  - [ ] Subtask 5.2: Component tests for TextCaptureInput
    - Test auto-focus and keyboard opening
    - Test save button enable/disable logic
    - Test haptic feedback trigger
    - Test animation on save
  - [ ] Subtask 5.3: Integration tests for text capture flow
    - Test end-to-end: tap button → type text → save
    - Test cancel flow with confirmation
    - Test offline text capture
    - Test empty text validation
  - [ ] Subtask 5.4: UX tests
    - Verify keyboard opens immediately (< 300ms)
    - Verify haptic feedback feels natural
    - Test multi-line text handling

## Dev Notes

### Architecture Context

**Bounded Context:** Capture Context (Supporting Domain)
**Aggregate:** `Capture` (polymorphic type='text')
**Domain Events:** `CaptureCreated` (same as audio)

**Text Capture Specifics:**
- No normalization needed (rawContent = normalizedText)
- Simpler flow than audio (no recording state)
- Instant capture vs. time-based recording

### Technical Stack

**Same as Story 2.1:**
- React Native + Expo
- WatermelonDB for local storage
- expo-haptics for feedback
- TypeScript strict mode

**Additional:**
- React Native TextInput (built-in component)
- Keyboard API for keyboard control
- Alert API for confirmation dialogs

### Performance Requirements

**NFR4:** Chargement liste < 1s (cache local)
**NFR6:** 0 capture perdue, jamais
**NFR7:** Disponibilité capture offline = 100%

### UX Requirements (Liquid Glass Design System)

- **Keyboard auto-open:** < 300ms
- **Haptic feedback:** Medium impact on save
- **Save animation:** Subtle fade + slide (60fps)
- **Multi-line support:** Auto-expand up to 5 lines, then scroll
- **Placeholder text:** Prompt créatif ("Notez votre pensée...")

### Reuse from Story 2.1

**Already implemented:**
- Capture WatermelonDB model (polymorphic type)
- Capture Context file structure
- Sync status tracking
- Offline queue mechanism

**Reusable components:**
- CaptureService base (extend for TextCaptureService)
- Sync queue integration

### File Structure

```
apps/mobile/
├── src/
│   ├── contexts/
│   │   └── Capture/
│   │       ├── domain/
│   │       │   └── Capture.model.ts  # Reuse from 2.1
│   │       ├── services/
│   │       │   ├── RecordingService.ts  # From 2.1
│   │       │   └── TextCaptureService.ts  # NEW
│   │       └── ui/
│   │           ├── RecordButton.tsx  # From 2.1
│   │           └── TextCaptureInput.tsx  # NEW
```

### Testing Standards

- Unit tests for TextCaptureService (validation, entity creation)
- Component tests for TextCaptureInput (keyboard, haptics, animation)
- Integration tests for full flow (tap → type → save)
- Offline mode tests
- UX performance tests (keyboard latency)

### Dependencies

**Already installed (from Story 2.1):**
- `@nozbe/watermelondb`
- `expo-haptics`
- React Native core (TextInput, Keyboard, Alert)

**No new dependencies required.**

### Previous Story Intelligence (Story 2.1)

**Learnings:**
- Capture model supports polymorphic types (audio/text)
- Offline-first pattern established with WatermelonDB
- Haptic feedback pattern established
- Sync queue mechanism functional

**Code patterns to reuse:**
- Capture entity creation flow
- SyncStatus tracking
- Service architecture pattern

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.2]
- [Source: _bmad-output/planning-artifacts/architecture.md#Capture Context]
- [Source: _bmad-output/implementation-artifacts/2-1-capture-audio-1-tap.md#Capture Model]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->
